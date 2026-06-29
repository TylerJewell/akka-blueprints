# SPEC — contract-review-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Contract Assistant (Multi-Agent).
**One-line pitch:** Upload a contract; a supervisor delegates clause analysis, risk scoring, and redline drafting to three specialist agents in parallel, then consolidates a reviewed package that a lawyer approves or rejects.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work to three AutonomousAgents in parallel, gathers their results, and asks a fourth AutonomousAgent to consolidate a reviewed contract package. The blueprint also demonstrates a **legal-sector sanitizer** that scrubs contract text before any agent sees it, a **human-in-the-loop approval gate** where a lawyer must accept or reject the redlines before the review is finalised, and a **before-agent-response guardrail** that vets the consolidator's output for prohibited clause suggestions.

## 3. User-facing flows

The user opens the App UI tab and submits a contract via the form (contract text or a reference name from the simulator's sample library).

1. The system creates a `ContractReview` record in `QUEUED` and enqueues a `ContractSubmitted` event.
2. The legal-sector sanitizer runs synchronously before any agent receives the text. If the contract contains prohibited content (e.g., clauses that mandate illegal discrimination), the review enters `REJECTED` immediately.
3. For clean contracts, the supervisor decomposes the contract into three parallel work items: a clause-extraction task for `ClauseAnalyst`, a risk-scoring task for `RiskScorer`, and a redline-drafting task for `Redliner`.
4. The workflow forks: all three agents run concurrently. Each returns a typed payload.
5. The supervisor merges the three payloads into a `ReviewPackage { clauseSummary, riskReport, redlines, guardrailVerdict }`.
6. A before-agent-response guardrail vets the consolidated package; if it fails, the review enters `BLOCKED`. Otherwise, the review moves to `AWAITING_APPROVAL`.
7. A lawyer calls `POST /api/reviews/{id}/approve` or `.../reject` to complete the HITL gate. On approval, the review enters `APPROVED` and the package is locked. On rejection, the review enters `REJECTED` with a lawyer note.
8. If any worker times out after 60 seconds, the workflow short-circuits: the supervisor consolidates from whichever outputs returned, and the review enters `DEGRADED`.

A `ContractSimulator` (TimedAction) drips a sample contract reference every 90 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReviewSupervisor` | `AutonomousAgent` | Decomposes the contract into a review plan; consolidates worker outputs; runs the output guardrail. | `ReviewWorkflow` | returns typed result to workflow |
| `ClauseAnalyst` | `AutonomousAgent` | Identifies and categorises key clauses (indemnity, liability cap, termination, IP assignment, etc.). | `ReviewWorkflow` | — |
| `RiskScorer` | `AutonomousAgent` | Assigns a risk level (LOW / MEDIUM / HIGH / CRITICAL) per clause and flags problematic language. | `ReviewWorkflow` | — |
| `Redliner` | `AutonomousAgent` | Drafts redline suggestions for HIGH and CRITICAL clauses. | `ReviewWorkflow` | — |
| `ReviewWorkflow` | `Workflow` | Orchestrates the parallel fan-out, the consolidation, the guardrail, and the HITL gate. | `ReviewEndpoint`, `ContractQueueConsumer` | `ContractReviewEntity` |
| `ContractReviewEntity` | `EventSourcedEntity` | Holds the review's lifecycle (queued → in-review → awaiting-approval → approved / rejected / degraded / blocked). | `ReviewWorkflow`, `ReviewEndpoint` | `ReviewView` |
| `ContractQueue` | `EventSourcedEntity` | Logs each submitted contract reference for replay/audit. | `ReviewEndpoint`, `ContractSimulator` | `ContractQueueConsumer` |
| `ReviewView` | `View` | List-of-reviews read model. | `ContractReviewEntity` events | `ReviewEndpoint` |
| `ContractQueueConsumer` | `Consumer` | Listens to `ContractQueue` events and starts a workflow per submission. | `ContractQueue` events | `ReviewWorkflow` |
| `ContractSimulator` | `TimedAction` | Drips a sample contract reference every 90 s. | scheduler | `ContractQueue` |
| `ReviewSampler` | `TimedAction` | Samples one approved review every 5 minutes for eval scoring; emits a `ReviewEvalScored` event. | scheduler | `ContractReviewEntity` |
| `ReviewEndpoint` | `HttpEndpoint` | `/api/reviews/*` — submit, get, list, approve, reject, SSE. | — | `ReviewView`, `ContractQueue`, `ContractReviewEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record ContractSubmission(String contractRef, String contractText, String submittedBy) {}

record ClauseEntry(String clauseId, String clauseType, String excerpt, String summary) {}
record ClauseSummary(List<ClauseEntry> clauses, Instant analysedAt) {}

record RiskFlag(String clauseId, RiskLevel riskLevel, String rationale) {}
record RiskReport(List<RiskFlag> flags, RiskLevel overallRisk, Instant scoredAt) {}

enum RiskLevel { LOW, MEDIUM, HIGH, CRITICAL }

record RedlineSuggestion(String clauseId, String originalText, String proposedText, String justification) {}
record RedlineSet(List<RedlineSuggestion> suggestions, Instant draftedAt) {}

record ReviewPlan(String clauseExtractionTask, String riskScoringTask, String redlineTask) {}

record ReviewPackage(ClauseSummary clauseSummary, RiskReport riskReport,
                     RedlineSet redlines, String guardrailVerdict, Instant consolidatedAt) {}

record ContractReview(
    String reviewId,
    String contractRef,
    String contractText,
    String submittedBy,
    ReviewStatus status,
    Optional<ClauseSummary> clauseSummary,
    Optional<RiskReport> riskReport,
    Optional<RedlineSet> redlines,
    Optional<ReviewPackage> reviewPackage,
    Optional<String> failureReason,
    Optional<String> lawyerNote,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReviewStatus { QUEUED, IN_REVIEW, AWAITING_APPROVAL, APPROVED, REJECTED, DEGRADED, BLOCKED }
```

### Events (on `ContractReviewEntity`)

`ReviewCreated`, `ReviewRejectedBySanitizer`, `ClauseSummaryAttached`, `RiskReportAttached`, `RedlineSetAttached`, `ReviewConsolidated`, `ReviewDegraded`, `ReviewBlocked`, `ReviewApproved`, `ReviewRejectedByLawyer`, `ReviewEvalScored`.

### Events (on `ContractQueue`)

`ContractSubmitted { reviewId, contractRef, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reviews` — body `{ contractRef, contractText, submittedBy }` → `{ reviewId }`. Starts a workflow.
- `GET /api/reviews` — list all reviews. Optional `?status=QUEUED|IN_REVIEW|AWAITING_APPROVAL|APPROVED|REJECTED|DEGRADED|BLOCKED`.
- `GET /api/reviews/{id}` — one review.
- `POST /api/reviews/{id}/approve` — body `{ lawyerNote }` → `{ reviewId, status: "APPROVED" }`. HITL approval gate.
- `POST /api/reviews/{id}/reject` — body `{ lawyerNote }` → `{ reviewId, status: "REJECTED" }`. HITL rejection gate.
- `GET /api/reviews/sse` — server-sent events stream of every review change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Contract Assistant (Multi-Agent)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a contract reference and text, live list of reviews with status pills, approve/reject buttons visible when status is AWAITING_APPROVAL, expand-row to see clause summary + risk report + redlines + eval score.

Browser title: `<title>Akka Sample: Contract Assistant (Multi-Agent)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — legal-sector sanitizer** (`sector` flavor): runs before any agent receives the contract text. Rejects contracts with prohibited discriminatory clauses or unlawful provisions. Blocking. Failure → `REJECTED`.
- **H1 — HITL approval gate** (`application` flavor): a lawyer must call `/approve` or `/reject` before the review moves to a terminal state from `AWAITING_APPROVAL`. Blocking. Implemented in `ReviewWorkflow.approvalStep` with a long stepTimeout (24 hours).
- **G1 — output guardrail** (`before-agent-response` on `ReviewSupervisor`): vets the consolidated review package for legally impermissible clause suggestions before the package is surfaced to the lawyer. Blocking. Failure → `BLOCKED`.

## 9. Agent prompts

- `ReviewSupervisor` → `prompts/review-supervisor.md`. Decomposes a contract into worker tasks; later consolidates clause summary, risk report, and redlines into a final package.
- `ClauseAnalyst` → `prompts/clause-analyst.md`. Identifies and categorises clauses; returns `ClauseSummary`.
- `RiskScorer` → `prompts/risk-scorer.md`. Assigns risk levels per clause; returns `RiskReport`.
- `Redliner` → `prompts/redliner.md`. Drafts redline suggestions for HIGH and CRITICAL clauses; returns `RedlineSet`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a contract; review progresses QUEUED → IN_REVIEW → AWAITING_APPROVAL within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `Redliner` timeout to 1 s); review enters DEGRADED with whichever partial outputs came back.
3. **J3** — Submit a contract with a prohibited clause; sanitizer rejects it immediately; review enters REJECTED before any agent runs.
4. **J4** — Inject a guardrail failure (supervisor returns an impermissible suggestion); review enters BLOCKED.
5. **J5** — Approve a review in AWAITING_APPROVAL; review moves to APPROVED; the package is locked in the entity.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named contract-review-team demonstrating the
delegation-supervisor-workers × legal-compliance cell. Runs out of the box
(no external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-legal-compliance-contract-review-team.
Java package io.akka.samples.contractassistantmultiagent. Akka 3.6.0.
HTTP port 9533.

Components to wire (exactly):
- 4 AutonomousAgents:
  * ReviewSupervisor — definition() with
    capability(TaskAcceptance.of(PLAN_REVIEW).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(CONSOLIDATE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/review-supervisor.md.
    Returns ReviewPlan{clauseExtractionTask, riskScoringTask, redlineTask}
    for PLAN_REVIEW and ReviewPackage{clauseSummary, riskReport, redlines,
    guardrailVerdict, consolidatedAt} for CONSOLIDATE.
  * ClauseAnalyst — capability(TaskAcceptance.of(EXTRACT_CLAUSES)
    .maxIterationsPerTask(3)). System prompt from prompts/clause-analyst.md.
    Returns ClauseSummary{clauses: List<ClauseEntry{clauseId, clauseType,
    excerpt, summary}>, analysedAt}.
  * RiskScorer — capability(TaskAcceptance.of(SCORE_RISK)
    .maxIterationsPerTask(2)). System prompt from prompts/risk-scorer.md.
    Returns RiskReport{flags: List<RiskFlag{clauseId, riskLevel, rationale}>,
    overallRisk, scoredAt}.
  * Redliner — capability(TaskAcceptance.of(DRAFT_REDLINES)
    .maxIterationsPerTask(3)). System prompt from prompts/redliner.md.
    Returns RedlineSet{suggestions: List<RedlineSuggestion{clauseId,
    originalText, proposedText, justification}>, draftedAt}.

- 1 Workflow ReviewWorkflow with steps:
  sanitizeStep -> planStep -> [parallel] extractStep, scoreStep, redlineStep
  -> joinStep -> consolidateStep -> guardrailStep -> approvalStep -> emitStep.
  sanitizeStep: deterministic legal-sector content filter; on failure, end
  with ReviewRejectedBySanitizer.
  planStep calls forAutonomousAgent(ReviewSupervisor.class, PLAN_REVIEW).
  extractStep, scoreStep, and redlineStep run in parallel (CompletionStage
  allOf); each governed by WorkflowSettings.builder()
  .stepTimeout(ReviewWorkflow::extractStep, ofSeconds(60))
  .stepTimeout(ReviewWorkflow::scoreStep, ofSeconds(60))
  .stepTimeout(ReviewWorkflow::redlineStep, ofSeconds(60)).
  On any timeout, transition to degradeStep that consolidates from available
  outputs and ends with ReviewDegraded.
  consolidateStep calls forAutonomousAgent(ReviewSupervisor.class,
  CONSOLIDATE) with merged inputs; give consolidateStep a 90s stepTimeout.
  guardrailStep runs the deterministic vetter + LLM judge on the consolidated
  package; on failure, end with ReviewBlocked.
  approvalStep is a waiting step with stepTimeout of 86400s (24 hours);
  it suspends until a lawyer calls /approve or /reject.

- 1 EventSourcedEntity ContractReviewEntity holding state ContractReview{
  reviewId, contractRef, contractText, submittedBy, ReviewStatus,
  Optional<ClauseSummary> clauseSummary, Optional<RiskReport> riskReport,
  Optional<RedlineSet> redlines, Optional<ReviewPackage> reviewPackage,
  Optional<String> failureReason, Optional<String> lawyerNote,
  Optional<Integer> evalScore, Optional<String> evalRationale,
  Instant createdAt, Optional<Instant> finishedAt}.
  ReviewStatus enum: QUEUED, IN_REVIEW, AWAITING_APPROVAL, APPROVED,
  REJECTED, DEGRADED, BLOCKED.
  Events: ReviewCreated, ReviewRejectedBySanitizer, ClauseSummaryAttached,
  RiskReportAttached, RedlineSetAttached, ReviewConsolidated, ReviewDegraded,
  ReviewBlocked, ReviewApproved, ReviewRejectedByLawyer, ReviewEvalScored.
  Commands: createReview, rejectBySanitizer, attachClauseSummary,
  attachRiskReport, attachRedlineSet, consolidate, degrade, block, approve,
  rejectByLawyer, recordEval, getReview.
  emptyState() returns ContractReview.initial("", "", "", "") with no
  commandContext() reference.

- 1 EventSourcedEntity ContractQueue with command enqueueContract(
  contractRef, contractText, submittedBy) emitting
  ContractSubmitted{reviewId, contractRef, submittedBy, submittedAt}.

- 1 View ReviewView with row type ContractReviewRow (mirrors ContractReview
  minus heavy nested payloads; every nullable field is Optional<T>). Table
  updater consumes ContractReviewEntity events. ONE query getAllReviews
  SELECT * AS reviews FROM review_view. No WHERE status filter (Akka cannot
  auto-index enum columns) — caller filters client-side.

- 1 Consumer ContractQueueConsumer subscribed to ContractQueue events; on
  ContractSubmitted starts a ReviewWorkflow with the reviewId as the workflow id.

- 2 TimedActions:
  * ContractSimulator — every 90s, reads next line from
    src/main/resources/sample-events/sample-contracts.jsonl and calls
    ContractQueue.enqueueContract.
  * ReviewSampler — every 5 minutes, queries ReviewView.getAllReviews, picks
    the oldest APPROVED review without an evalScore, runs a 1–5 rubric
    judge over the review package, then calls
    ContractReviewEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * ReviewEndpoint at /api with POST /reviews, GET /reviews,
    GET /reviews/{id}, POST /reviews/{id}/approve, POST /reviews/{id}/reject,
    GET /reviews/sse, and the /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ReviewTasks.java declaring four Task<R> constants: PLAN_REVIEW (ReviewPlan),
  EXTRACT_CLAUSES (ClauseSummary), SCORE_RISK (RiskReport), DRAFT_REDLINES
  (RedlineSet), CONSOLIDATE (ReviewPackage).
- Domain records ReviewPlan, ClauseEntry, ClauseSummary, RiskFlag, RiskReport,
  RiskLevel enum, RedlineSuggestion, RedlineSet, ReviewPackage.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9533 and akka.javasdk.agent model-provider
  blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-contracts.jsonl with 8 canned
  contract reference lines (NDA, MSA, SoW, license agreement, etc.).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (S1 sector sanitizer,
  H1 HITL application approval, G1 output guardrail before-agent-response)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  sector.primary = legal-services, decisions.authority_level = recommend-only,
  data.data_classes.pii = true (contracts contain party names),
  capabilities.legal_reasoning = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/review-supervisor.md, prompts/clause-analyst.md,
  prompts/risk-scorer.md, prompts/redliner.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Contract Assistant
  (Multi-Agent)", one-line pitch, prerequisites, generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs: Overview, Architecture
  (4 mermaid diagrams + click-to-expand component table with syntax-highlighted
  Java snippets), Risk Survey (7 sub-tabs from governance.html; unanswered .qb
  opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/
  Source table with click-to-expand rows), App UI (form + live list with status
  pills, approve/reject buttons when status is AWAITING_APPROVAL).
  Browser title exactly:
  <title>Akka Sample: Contract Assistant (Multi-Agent)</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (review-supervisor.json,
  clause-analyst.json, risk-scorer.json, redliner.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return
  shape.
- Per-agent mock-response shapes for THIS blueprint:
    review-supervisor.json — list of either ReviewPlan or ReviewPackage
      objects. 4–6 ReviewPlan entries (clauseExtractionTask + riskScoringTask
      + redlineTask triples) and 4–6 ReviewPackage entries (each with a
      ClauseSummary, RiskReport, RedlineSet, guardrailVerdict = "ok").
    clause-analyst.json — 4–6 ClauseSummary entries, each with 3–6 clauses
      covering types like "indemnity", "limitation-of-liability", "termination",
      "ip-assignment", "governing-law".
    risk-scorer.json — 4–6 RiskReport entries, each with 3–6 RiskFlag objects
      and an overallRisk value.
    redliner.json — 4–6 RedlineSet entries, each with 2–4 RedlineSuggestion
      objects targeting HIGH or CRITICAL clauses.
- A MockModelProvider.seedFor(reviewId) helper makes the selection
  deterministic per review id so the same contract produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause
  matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit
  stepTimeout (60s workers, 90s consolidation, 86400s approval gate); default
  5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion ReviewTasks.java declaring
  Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9533 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc). Without these, state names
  render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  NodeList index; delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage allOf, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, marketing tone.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
