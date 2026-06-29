# Lead Score HITL — Specification

This is the natural-language input for `/akka:specify`. Run `/akka:specify @SPEC.md` from inside this folder. Sections 1–11 together are the spec; Section 12 auto-chains the rest.

---

## 1. System name + one-line pitch

**Lead Score HITL.** Inbound leads are collected, analyzed, and scored by agents; a human approves the top-ranked leads; outreach email is then drafted for the approved leads.

## 2. What this blueprint demonstrates

The coordination pattern is human-in-loop-gate: four agents run a lead pipeline, and a human decision gate sits between scoring and outreach so no email is drafted for a lead a person has not approved. The governance pattern wires three controls — a human approval gate (hitl, application flavor), PII sanitization of lead data before storage and before any model call (sanitizer, pii flavor), and a before-agent-response guardrail on the outreach agent that blocks off-policy claims before an email leaves the system.

## 3. User-facing flows

1. A lead arrives — submitted via `POST /api/leads` or dripped by `LeadSimulator`. The system records it in `NEW` and starts a `LeadScoringWorkflow`.
2. The workflow collects and enriches the lead (`LeadCollectionAgent`), sanitizing PII first; the lead moves to `COLLECTED`.
3. The workflow analyzes the lead (`LeadAnalysisAgent`); the lead moves to `ANALYZED`.
4. The workflow scores the lead 0–100 (`LeadScoringAgent`); the lead moves to `SCORED` and the workflow waits.
5. `LeadRankingMonitor` periodically marks the top-ranked `SCORED` leads as `SHORTLISTED`; they surface in the UI with Approve/Reject buttons.
6. A reviewer approves or rejects a shortlisted lead. Approve moves it to `APPROVED`; reject moves it to a terminal `REJECTED`.
7. On approval the workflow resumes, `OutreachAgent` drafts an outreach email (guardrail-checked), and the lead moves to `CONTACTED` with a draft subject and body.

These become the acceptance journeys in `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `LeadCollectionAgent` | AutonomousAgent | Enriches a sanitized lead into a profile | `LeadScoringWorkflow` | `LeadEntity` |
| `LeadAnalysisAgent` | AutonomousAgent | Summarizes fit and intent | `LeadScoringWorkflow` | `LeadEntity` |
| `LeadScoringAgent` | AutonomousAgent | Produces a 0–100 score + rationale | `LeadScoringWorkflow` | `LeadEntity` |
| `OutreachAgent` | AutonomousAgent | Drafts an outreach email | `LeadScoringWorkflow` | `LeadEntity` |
| `LeadScoringWorkflow` | Workflow | Orchestrates collect → analyze → score → await-review → outreach | `InboundLeadConsumer` | the four agents, `LeadEntity` |
| `LeadEntity` | EventSourcedEntity | Per-lead durable lifecycle | workflow, endpoint | `LeadsView` |
| `InboundLeadQueue` | EventSourcedEntity | Records each inbound lead submission | `LeadSimulator`, `LeadEndpoint` | `InboundLeadConsumer` |
| `LeadsView` | View | List read model, SSE source | `LeadEntity` events | `LeadEndpoint`, `LeadRankingMonitor` |
| `InboundLeadConsumer` | Consumer | Starts a workflow per inbound lead | `InboundLeadQueue` events | `LeadScoringWorkflow` |
| `LeadSimulator` | TimedAction | Drips sample leads every 30s | sample-events file | `InboundLeadQueue` |
| `LeadRankingMonitor` | TimedAction | Shortlists top-ranked SCORED leads every 30s | `LeadsView` | `LeadEntity` |
| `LeadEndpoint` | HttpEndpoint | `/api` surface + SSE + metadata | clients | entities, view |
| `AppEndpoint` | HttpEndpoint | Serves `/` redirect and `/app/*` | browser | static-resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions on startup | — | — |

Component count: 4 autonomous-agent · 1 workflow · 2 event-sourced-entity · 1 view · 1 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup.

## 5. Data model

Full detail in `reference/data-model.md`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

```java
record Lead(
  String id,
  Optional<String> company,
  Optional<String> contactName,        // PII — stored only after sanitization
  Optional<String> rawSource,
  LeadStatus status,
  Optional<Instant> collectedAt,
  Optional<String>  enrichedProfile,
  Optional<Instant> analyzedAt,
  Optional<String>  analysisSummary,
  Optional<Instant> scoredAt,
  Optional<Integer> score,
  Optional<String>  scoreRationale,
  Optional<Boolean> shortlisted,
  Optional<Instant> reviewedAt,
  Optional<String>  reviewedBy,
  Optional<String>  reviewDecision,    // APPROVED | REJECTED
  Optional<String>  reviewComment,
  Optional<Instant> outreachDraftedAt,
  Optional<String>  outreachSubject,
  Optional<String>  outreachBody
) {
  static Lead initial(String id, String company) { /* all Optional.empty() */ }
  Lead applyEvent(LeadEvent e);
}
```

`LeadStatus` enum: `NEW · COLLECTED · ANALYZED · SCORED · SHORTLISTED · APPROVED · REJECTED · CONTACTED`.

`LeadEvent` sealed interface, variants: `LeadCollected`, `LeadAnalyzed`, `LeadScored`, `LeadShortlisted`, `LeadApproved`, `LeadRejected`, `OutreachDrafted`.

Auxiliary records: `CollectedLead(String company, String contactName, String enrichedProfile)`, `LeadAnalysis(String summary)`, `LeadScore(int score, String rationale)`, `OutreachEmail(String subject, String body)`, `ReviewDecision(String reviewedBy, String decision, String comment)`.

## 6. API contract

Full schemas in `reference/api-contract.md`. Surface:

```
POST /api/leads                      -> { leadId }
POST /api/leads/{leadId}/approve     -> 200 | 404
POST /api/leads/{leadId}/reject      -> 200 | 404
GET  /api/leads ?status=...          -> { leads: [Lead, ...] }
GET  /api/leads/{leadId}             -> Lead
GET  /api/leads/sse                  -> Server-Sent Events of Lead
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/`, no npm). Browser title `<title>Akka Sample: Lead Score HITL</title>`. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index, with no hidden zombie panels (Lesson 26). Mermaid diagrams on the Architecture tab use the Akka theme variables plus the CSS overrides for state labels and edge-label `foreignObject` overflow (Lesson 24). The App UI tab submits a lead, streams the live lead list via SSE, and shows Approve/Reject buttons on `SHORTLISTED` leads. Full description in `reference/ui-mockup.md`.

## 8. Governance

Controls in `eval-matrix.yaml`; deployer survey in `risk-survey.yaml`.

- **H1 — hitl · application.** `LeadScoringWorkflow` pauses in `awaitReviewStep` until a reviewer approves or rejects a shortlisted lead via `POST /api/leads/{id}/approve|reject`.
- **S1 — sanitizer · pii.** `PiiSanitizer` redacts and normalizes contact PII inside `collectStep` before the lead is stored and before any agent prompt is built; no raw PII reaches a model call.
- **G1 — guardrail · before-agent-response.** A before-agent-response guardrail on `OutreachAgent` blocks off-policy claims (pricing promises, unsupported guarantees) before the drafted email is persisted.

## 9. Agent prompts

One file per agent under `prompts/`:

- `prompts/lead-collection-agent.md` — enrich a sanitized lead into a structured profile.
- `prompts/lead-analysis-agent.md` — summarize fit and buying intent.
- `prompts/lead-scoring-agent.md` — assign a 0–100 score with a short rationale.
- `prompts/outreach-agent.md` — draft a short, on-policy outreach email.

## 10. Acceptance

Full journeys in `reference/user-journeys.md`. Inlined:

1. **Submit and score.** `POST /api/leads` returns a `leadId`; within ~60s the lead reaches `SCORED` with a non-empty `score` and `scoreRationale` over SSE.
2. **Shortlist.** A high-scoring lead is marked `SHORTLISTED` by `LeadRankingMonitor` and surfaces with Approve/Reject controls.
3. **Approve and draft outreach.** Approve drives the lead to `CONTACTED` with a non-empty `outreachSubject` and `outreachBody` within ~60s.
4. **Reject.** Reject with a reason drives the lead to terminal `REJECTED`; no outreach is drafted.

## 11. Implementation directives

```
Create a sample named lead-score-hitl demonstrating the human-in-loop-gate ×
sales-marketing cell. Runs out of the box (no external services). Maven group
io.akka.samples, artifact lead-score-hitl. Java package
io.akka.samples.leadscorehitl. Akka 3.6.0. HTTP port 9178.

Components to wire (exactly):
- 4 AutonomousAgents, each with definition() declaring
  capability(TaskAcceptance.of(task).maxIterationsPerTask(3)):
  - LeadCollectionAgent  -> CollectedLead(company, contactName, enrichedProfile)
  - LeadAnalysisAgent    -> LeadAnalysis(summary)
  - LeadScoringAgent     -> LeadScore(score, rationale)
  - OutreachAgent        -> OutreachEmail(subject, body)
- 1 Workflow LeadScoringWorkflow with steps collectStep -> analyzeStep ->
  scoreStep -> awaitReviewStep -> outreachStep. Each agent step calls
  forAutonomousAgent(Agent.class, role+leadId).runSingleTask(...) then
  forTask(taskId).result(...). collectStep runs PiiSanitizer over the raw lead
  BEFORE building the agent prompt and BEFORE LeadEntity.recordCollected.
  awaitReviewStep polls LeadEntity.getLead; on SCORED or SHORTLISTED it
  self-schedules a 5-second resume timer and pauses; on APPROVED it transitions
  to outreachStep; on REJECTED it ends. Override settings() with
  stepTimeout(60s) on collectStep, analyzeStep, scoreStep, outreachStep and
  stepTimeout(10s) on awaitReviewStep. WorkflowSettings is nested in Workflow —
  no import.
- 1 EventSourcedEntity LeadEntity holding a Lead record with id, company
  (Optional), contactName (Optional), rawSource (Optional), LeadStatus enum,
  and the Optional lifecycle fields from Section 5. Events: LeadCollected,
  LeadAnalyzed, LeadScored, LeadShortlisted, LeadApproved, LeadRejected,
  OutreachDrafted. Commands: recordCollected, recordAnalysis, recordScore,
  markShortlisted, approve, reject, recordOutreach, getLead. emptyState()
  returns Lead.initial("", "") with no commandContext() reference.
- 1 EventSourcedEntity InboundLeadQueue (single instance "default") with one
  command enqueueLead(rawSource) emitting InboundLeadQueued.
- 1 View LeadsView with row type Lead, table updater consuming LeadEntity
  events. ONE query getAllLeads: SELECT * AS leads FROM leads_view. No WHERE
  status filter (Akka cannot auto-index enum columns) — filter client-side.
  Provide streamAllLeads for SSE.
- 1 Consumer InboundLeadConsumer subscribed to InboundLeadQueue events; on each
  event starts a LeadScoringWorkflow with a fresh UUID.
- 2 TimedActions: LeadSimulator (every 30s, reads next line from
  src/main/resources/sample-events/inbound-leads.jsonl and calls
  InboundLeadQueue.enqueueLead); LeadRankingMonitor (every 30s, queries
  LeadsView.getAllLeads, ranks SCORED leads by score desc, marks the top 3 as
  SHORTLISTED via LeadEntity.markShortlisted).
- 2 HttpEndpoints: LeadEndpoint at /api with leads submit, approve, reject,
  leads list (filter client-side from getAllLeads), single lead, SSE stream,
  and three /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- LeadScoringTasks.java declaring four Task<R> constants: COLLECT
  (resultConformsTo CollectedLead), ANALYZE (LeadAnalysis), SCORE (LeadScore),
  OUTREACH (OutreachEmail).
- CollectedLead(String company, String contactName, String enrichedProfile),
  LeadAnalysis(String summary), LeadScore(int score, String rationale),
  OutreachEmail(String subject, String body), ReviewDecision(String reviewedBy,
  String decision, String comment).
- PiiSanitizer.java — a static helper redacting emails/phones and normalizing
  contact names; called from collectStep and reused before any agent prompt.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9178 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/inbound-leads.jsonl with 8 canned leads.
- src/main/resources/metadata/{eval-matrix.yaml, risk-survey.yaml, README.md}
  (copies of the root files for the endpoint to serve from the classpath).
- eval-matrix.yaml at the project root with the 3 controls (H1, S1, G1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types (includes contact PII), capability.*, model.*, subjects.children;
  marking deployer fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root per the authoring guide. NO governance-
  mechanisms section, NO configuration section.
- src/main/resources/static-resources/index.html — single self-contained HTML
  file. Five tabs (Overview / Architecture / Risk Survey / Eval Matrix /
  App UI). Match the governance.html visual style (dark / yellow / Instrument
  Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (LeadCollectionAgent -> CollectedLead; LeadAnalysisAgent ->
  LeadAnalysis; LeadScoringAgent -> LeadScore; OutreachAgent -> OutreachEmail;
  write 4-6 entries each to src/main/resources/mock-responses/<agent>.json).
  Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Constraints — see AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: workflow step timeouts set explicitly on every agent-calling step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: every AutonomousAgent has its Task<R> constant in LeadScoringTasks.
- Lesson 8: verify model names current before locking them in.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port (9178).
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column with no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label colour + edge-label overflow CSS overrides.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute, no zombie panels.
- emptyState() never calls commandContext().
```

## 12. Post-scaffolding workflow

When `/akka:specify` finishes scaffolding the specification into `specs/features/`, do not stop and wait for the user. Continue automatically:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option when ambiguous, consistent with Sections 4 and the PLAN.
2. Run `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and summarise all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

When `/akka:build` reports the service is running, output just the listening URL (`http://localhost:9178`) and a one-line summary of any failures from step 3. Stop earlier only on a hard error you cannot work around without the user — an unresolved API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
