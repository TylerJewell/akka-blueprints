# Blackboard Knowledge Discovery — SPEC

## 1. System name + pitch
**System name:** Blackboard Knowledge Discovery
**Pitch:** Submit a research question; a roster of specialist analyst agents read and write to a shared structured knowledge object — the blackboard — while a control shell decides which specialist runs next based on what the blackboard currently holds. No specialist messages another; coordination is mediated entirely through reads and writes to the shared object.

## 2. What this blueprint demonstrates
The **blackboard** coordination pattern with Akka primitives: a `ControlShell` workflow loops over an inquiry's current `BlackboardEntity` state, selects the next specialist to invoke (source-finding, claim-extraction, hypothesis-synthesis, critique), runs the chosen `AutonomousAgent`, and commits its proposed write back to the entity. The entity's single-writer guarantee serialises writes; a before-state-write schema guardrail rejects malformed contributions; an on-decision-eval event scores each accepted write against a contribution-utility metric so the control shell can stop invoking specialists that no longer help.

## 3. User-facing flows
1. User submits a research question via the App UI form
2. System logs the submission to `IntakeQueue` and starts an inquiry lifecycle on `InquiryEntity`
3. `InquiryRequestConsumer` picks up the submission and starts the `ControlShell` workflow for that inquiry
4. `ControlShell` inspects the inquiry's `BlackboardEntity` state and picks the next specialist
5. Selected specialist (e.g., `SourceScout`) runs and proposes a write
6. Write passes through a before-state-write schema guardrail; rejected writes do not touch the entity
7. Accepted writes append to the blackboard; an on-decision-eval scores the contribution's utility
8. `ControlShell` loops until either every open question has supporting claims and a non-disputed hypothesis exists, or the iteration budget is reached
9. `Critic` runs periodically; when it flags an item, the item moves to the blackboard's `disputed` list and the control shell reroutes
10. When the inquiry converges, `InquiryEntity` records the report
11. `InquirySimulator` drips a canned research question every 90 s; `StaleInquiryMonitor` closes inquiries idle more than 5 min
12. Operator can halt the whole shell via `SystemControl`

## 4. Components

| Component | Primitive | Role |
|---|---|---|
| ControlShell | Workflow | Inspect blackboard, choose next specialist, commit writes |
| SourceScout | AutonomousAgent | Propose new sources for the open question |
| ClaimExtractor | AutonomousAgent | Turn a source into structured claims |
| HypothesisSynthesizer | AutonomousAgent | Propose a hypothesis from current claims |
| Critic | AutonomousAgent | Challenge weakest claim or hypothesis |
| BlackboardEntity | EventSourcedEntity | One per inquiry; the shared structured state |
| InquiryEntity | EventSourcedEntity | Inquiry lifecycle |
| IntakeQueue | EventSourcedEntity | Submission audit log |
| SystemControl | KeyValueEntity | Operator halt flag |
| BlackboardView | View | Read-side projection of every inquiry's blackboard |
| InquiryRequestConsumer | Consumer | Subscribes to IntakeQueue; starts ControlShell |
| ContributionScorer | Consumer | Subscribes to BlackboardEntity; emits on-decision-eval records |
| InquirySimulator | TimedAction | Drip canned questions every 90 s |
| StaleInquiryMonitor | TimedAction | Close inquiries idle more than 5 min |
| KnowledgeEndpoint | HttpEndpoint | `/api/*` |
| AppEndpoint | HttpEndpoint | Serve embedded UI |
| Bootstrap | service-setup | Schedule TimedActions; ensure one ControlShell per active inquiry |

## 5. Data model

**Records:**
- `ResearchQuestion(inquiryId, question, submittedBy)`
- `Source(sourceId, citation, url, addedAt)`
- `Claim(claimId, text, supports: List<String>, derivedFrom: String, confidence: double)`
- `Hypothesis(hypothesisId, statement, backedByClaims: List<String>, proposedAt)`
- `Challenge(challengeId, targetId, targetKind, rationale, raisedAt)`
- `ProposedWrite(specialist, writeKind, payload)`
- `ContributionScore(score, rationale, scoredAt)`
- `ReportSummary(inquiryId, hypothesis, supportingClaims, openQuestions, summarisedAt)`

**Entity state — `Blackboard`:**
```java
record Blackboard(
    String inquiryId,
    String question,
    BlackboardStatus status,
    List<Source> sources,
    List<Claim> claims,
    List<Hypothesis> hypotheses,
    List<Challenge> disputed,
    List<String> openSubQuestions,
    int iterationCount,
    Optional<ReportSummary> report,
    Optional<Instant> lastWriteAt,
    Instant createdAt
) {}

enum BlackboardStatus { OPEN, CONVERGED, CLOSED }
```

**Entity state — `Inquiry`:**
```java
record Inquiry(
    String inquiryId,
    String question,
    String submittedBy,
    InquiryStatus status,
    Optional<String> reportSummary,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum InquiryStatus { INTAKE, ACTIVE, CONVERGED, CLOSED }
```

**Events:**
- BlackboardEntity: SourceAdded, ClaimAdded, HypothesisProposed, ChallengeRaised, SubQuestionOpened, SubQuestionAnswered, WriteRejected, IterationAdvanced, Converged, Closed
- InquiryEntity: InquiryCreated, InquiryActivated, InquiryConverged, InquiryClosed, ReportRecorded
- IntakeQueue: InquirySubmitted
- SystemControl: state-driven (no events)

## 6. API contract

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/inquiries` | `{ question, submittedBy? }` | `202 { inquiryId }` | KnowledgeEndpoint → IntakeQueue |
| GET | `/api/inquiries/{id}` | — | `200 Inquiry` / `404` | KnowledgeEndpoint ← InquiryEntity |
| GET | `/api/blackboards/{id}` | — | `200 Blackboard` / `404` | KnowledgeEndpoint ← BlackboardEntity |
| GET | `/api/blackboards` | — | `200 [ BlackboardRow... ]` | KnowledgeEndpoint ← BlackboardView |
| GET | `/api/blackboards?status=…` | — | `200 [ BlackboardRow... ]` (client-side filter) | KnowledgeEndpoint ← BlackboardView |
| GET | `/api/blackboards/sse` | — | `text/event-stream` | KnowledgeEndpoint ← BlackboardView |
| POST | `/api/inquiries/{id}/close` | `{ reason }` | `200 Inquiry` | KnowledgeEndpoint → InquiryEntity |
| POST | `/api/control/halt` | `{ reason, by }` | `200 SystemControl` | KnowledgeEndpoint → SystemControl |
| POST | `/api/control/resume` | — | `200 SystemControl` | KnowledgeEndpoint → SystemControl |
| GET | `/api/control` | — | `200 SystemControl` | KnowledgeEndpoint ← SystemControl |
| GET | `/api/metadata/readme` | — | `text/markdown` | AppEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | AppEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | AppEndpoint |
| GET | `/` | — | `302 → /app/index.html` | AppEndpoint |
| GET | `/app/*` | — | static UI (single self-contained index.html) | AppEndpoint |

## 7. UI
Five-tab structure: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Browser title: "Akka Sample: Blackboard Knowledge Discovery".

**Overview:** eyebrow + headline (no subtitle); cards: Try it / How it works / Components / API contract
**Architecture:** component graph, sequence, blackboard state machine, entity model (mermaid with Akka theme + Lesson 24 CSS overrides); per-component table with click-to-expand Java snippets
**Risk Survey:** seven sections rendered from `risk-survey.yaml`; deployer-specific fields faded
**Eval Matrix:** 5-column table (ID/Control/Mechanism/Implementation/Source) with click-to-expand rationale + implementation; colored mechanism pills
**App UI:** question form, live blackboard panel for each active inquiry (sources / claims / hypotheses / disputed / open sub-questions / report), Halt/Resume control

## 8. Governance
- **G1 — before-state-write schema guardrail** (guardrail, blocking): every proposed write to a blackboard is schema-validated; malformed writes are recorded as `WriteRejected` and never applied to entity state
- **E1 — on-decision-eval contribution-utility scoring** (eval-event, advisory): each accepted write is scored by `ContributionScorer` against a contribution-utility metric; the control shell consults recent scores when picking the next specialist

## 9. Agent prompts
- `SourceScout` → `prompts/source-scout.md`
- `ClaimExtractor` → `prompts/claim-extractor.md`
- `HypothesisSynthesizer` → `prompts/hypothesis-synthesizer.md`
- `Critic` → `prompts/critic.md`

## 10. Acceptance
See `reference/user-journeys.md`:
1. **J1** — Submit → control shell runs specialists → blackboard converges → report rendered
2. **J2** — Schema-violating write → guardrail rejects, `WriteRejected` recorded
3. **J3** — Low-utility contributions → scorer records low scores, control shell stops invoking that specialist
4. **J4** — Critic challenge → item moves to `disputed`, control shell reroutes
5. **J5** — Operator halt → shell pauses; resume continues the inquiry

## 11. Implementation directives
Folder: `blackboard.research-intel.shared-knowledge-discovery`
Maven group: `io.akka.samples`
Maven artifact: `blackboard-research-intel-shared-knowledge-discovery`
Java package: `io.akka.samples.blackboardknowledgediscovery`
Akka: 3.6.0
HTTP port: 9876

Components to wire: 4 AutonomousAgents (SourceScout, ClaimExtractor, HypothesisSynthesizer, Critic — each with the before-state-write schema guardrail applied at write commit time), 1 Workflow (ControlShell with 90s step timeout on the specialist-invocation step), 3 EventSourcedEntities (Blackboard, Inquiry, IntakeQueue), 1 KeyValueEntity (SystemControl), 1 View (BlackboardView), 2 Consumers (InquiryRequestConsumer, ContributionScorer), 2 TimedActions (InquirySimulator, StaleInquiryMonitor), 2 HttpEndpoints, 1 service-setup Bootstrap.

**Key constraints (Lessons applied):**
- Run command: `/akka:build` — not raw mvn (Lesson 1)
- `Optional<T>` for every nullable View row field; never primitive null in projections (Lesson 6)
- `WorkflowSettings` nested inside the `Workflow` (Lesson 7)
- `emptyState()` never calls `commandContext()` (Lesson 8)
- `AutonomousAgent` never silently downgraded to plain `Agent` (Lesson 9)
- View has no WHERE filter on enum status; filter client-side (Lesson 10)
- Explicit 90s `stepTimeout` on the agent-calling step (Lesson 11)
- Model names verified current before locking `application.conf` (Lesson 12)
- Explicit `http-port = 9876` in `application.conf` (Lesson 13)
- Mermaid theme variables AND CSS overrides for state labels (Lesson 24)
- Tab switching by `data-tab`/`data-panel` attributes, never by NodeList index (Lesson 26)
- Single self-contained `index.html` under `static-resources/` (Lesson 25)
- Try-it card shows just `/akka:build` (Lesson 23) — no env-var export block
- No forbidden words (see writing-style appendix in the corpus)
- Lessons 4 and 5: per-step timeout configured explicitly; deterministic step IDs used for resumption

**Mock LLM option:**
- Generate `MockModelProvider` dispatching on agent class
- `source-scout.json`: 5 entries proposing realistic sources for canned inquiries
- `claim-extractor.json`: 5 entries with structured claims of varying confidence
- `hypothesis-synthesizer.json`: 4 entries; one deliberately malformed to exercise G1
- `critic.json`: 4 entries; one flagging a hypothesis to drive J4
- `MockModelProvider.seedFor(inquiryId)` for deterministic per-inquiry output

**Metadata files:**
- `eval-matrix.yaml` with 2 controls (G1, E1), no `regulation_anchors`
- `risk-survey.yaml` pre-filling sector, decisions, data classes, capabilities, model family, oversight from the `research-intel` domain; deployer-specific fields marked `TO_BE_COMPLETED_BY_DEPLOYER`
- `prompts/source-scout.md`, `prompts/claim-extractor.md`, `prompts/hypothesis-synthesizer.md`, `prompts/critic.md`
- `README.md` at root (no Configuration or governance-mechanisms section)
- `src/main/resources/application.conf` with `akka.javasdk.dev-mode.http-port = 9876` and model-provider blocks
- `src/main/resources/sample-events/research-questions.jsonl` with 6 canned inquiries
- `src/main/resources/metadata/` copies of `eval-matrix.yaml`, `risk-survey.yaml`, `README.md`
- `src/main/resources/static-resources/index.html` (single self-contained file)

## 12. Post-scaffolding workflow — auto-chain
When `/akka:specify` finishes scaffolding:
1. Run `/akka:plan` (accept defaults)
2. Run `/akka:tasks` (accept defaults)
3. Run `/akka:implement` (run all tasks; capture failures, continue; summary at end)
4. Run `/akka:build`

Output just the listening URL and a one-line summary of failures. Stop early only for hard errors (missing API key, unrecoverable compile error, network failure).

## 13. Lessons applied
This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
