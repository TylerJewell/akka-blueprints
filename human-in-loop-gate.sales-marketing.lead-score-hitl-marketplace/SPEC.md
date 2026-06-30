# SPEC — lead-score-flow

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Lead Score Flow.
**One-line pitch:** A user submits a lead profile; `ScoringAgent` scores it against ICP criteria; the workflow pauses at a human review gate for ambiguous scores; a reviewer approves or rejects through the API; on approval `QualificationAgent` generates the qualification summary and next steps.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the sales-marketing domain: a 3-task graph that scores a lead, then waits at an unassigned review task that a person completes through the API, then generates a qualification summary only if the reviewer approves. The governance pattern includes an application-level human approval gate between the score and qualify phases, an eval-event emitted on each scoring decision for drift detection, and a tool-permission guardrail that blocks the qualify step unless the lead is approved.

## 3. User-facing flows

1. A client POSTs a lead profile to `/api/score-request`. The response returns `{ leadId }`. The lead appears in the UI in `SCORED` once `ScoringAgent` finishes (typically 5–30 s), with the score, rationale, and confidence visible.
2. The reviewer reads the score and clicks Approve. This POSTs to `/api/leads/{leadId}/approve`. The workflow resumes, `QualificationAgent` runs, and the qualification summary with next steps appears with status `QUALIFIED`.
3. The reviewer clicks Reject with a reason. This POSTs to `/api/leads/{leadId}/reject`. The lead moves to terminal `DISQUALIFIED` and the reason is shown. The qualify step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ScoringAgent | AutonomousAgent | Scores a lead against ICP criteria; returns `LeadScore{score, rationale, confidence}` | LeadScoringWorkflow | LeadEntity |
| QualificationAgent | AutonomousAgent | Generates qualification summary after approval; returns `QualificationSummary{verdict, nextSteps}` | LeadScoringWorkflow | LeadEntity |
| LeadScoringWorkflow | Workflow | Orchestrates score → await review → qualify | LeadScoringEndpoint | ScoringAgent, QualificationAgent, LeadEntity |
| LeadEntity | EventSourcedEntity | Holds the lead state and lifecycle events | LeadScoringWorkflow, LeadScoringEndpoint | LeadsView |
| LeadsView | View | CQRS read model of all leads | LeadEntity | LeadScoringEndpoint |
| LeadScoringEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | LeadScoringWorkflow, LeadEntity, LeadsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Lead` (LeadEntity state and LeadsView row): `id` (String), `companyName` (`Optional<String>`), `contactEmail` (`Optional<String>`), `status` (LeadStatus enum), and lifecycle fields all `Optional<T>`: `scoredAt`, `score`, `scoreRationale`, `scoreConfidence`, `approvedAt`, `approvedBy`, `approverNote`, `rejectedAt`, `rejectedBy`, `rejectReason`, `qualifiedAt`, `qualificationVerdict`, `qualificationNextSteps`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`LeadStatus` enum: `SCORED`, `APPROVED`, `DISQUALIFIED`, `QUALIFIED`.

Events: `LeadScored`, `LeadApproved`, `LeadRejected`, `LeadQualified`.

Domain records: `LeadScore(int score, String rationale, String confidence)`, `ReviewDecision(String approvedBy, String note)`, `QualificationSummary(String verdict, String nextSteps)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/score-request              -> { leadId }
POST /api/leads/{leadId}/approve     -> 200 | 404
POST /api/leads/{leadId}/reject      -> 200 | 404
GET  /api/leads                      -> { leads: [Lead, ...] }
GET  /api/leads/{leadId}             -> Lead
GET  /api/leads/sse                  -> Server-Sent Events of Lead
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: Lead Score Flow</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a lead profile, lists leads live via SSE, and shows Approve/Reject buttons on `SCORED` leads that have a rationale. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `LeadScoringWorkflow` pauses at the await-review task; `/api/leads/{id}/approve` and `/api/leads/{id}/reject` resume it.
- **E1 — eval-event · on-decision-eval.** An eval event is emitted after each `LeadScored` event, recording the score, confidence, and the input profile for drift analysis.
- **G1 — guardrail · before-tool-call.** A guardrail on `QualificationAgent` verifies `LeadEntity.status == APPROVED` before the qualification tool runs.

## 9. Agent prompts

- `ScoringAgent` — scores a lead against ICP criteria and returns a score with rationale. See `prompts/scoring-agent.md`.
- `QualificationAgent` — generates a qualification summary for an approved lead. See `prompts/qualification-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Score a lead.** POST a lead profile; within ~30 s a lead appears in `SCORED` with non-null `score`, `scoreRationale`, and `scoreConfidence`.
2. **Approve and qualify.** Approve a `SCORED` lead; it reaches `QUALIFIED` with non-null `qualificationVerdict` and `qualificationNextSteps` within ~30 s.
3. **Reject a lead.** Reject a `SCORED` lead with a reason; it moves to terminal `DISQUALIFIED` and the reason shows.
4. **Qualify guard.** The qualify step is never reached for a lead that is not `APPROVED`.
5. **Eval event.** Each scoring decision emits an eval event visible in the event log endpoint.

---

## 11. Implementation directives

```
Create a sample named lead-score-flow demonstrating the human-in-loop-gate ×
sales-marketing cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-sales-marketing-lead-score-hitl-marketplace.
Java package io.akka.samples.leadscoreflow. Akka 3.6.0. HTTP port 9620.

Components to wire (exactly):
- 2 AutonomousAgents: ScoringAgent (scores a lead against ICP criteria, returns a
  typed LeadScore{score,rationale,confidence}) and QualificationAgent (returns a typed
  QualificationSummary{verdict,nextSteps}). Each declares definition() returning an
  AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow LeadScoringWorkflow with three tasks: scoreStep -> awaitReviewStep
  -> qualifyStep. scoreStep calls forAutonomousAgent(ScoringAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordScore on
  LeadEntity, and emits an eval event. awaitReviewStep polls LeadEntity.getLead;
  on SCORED it self-schedules a 5-second resume timer; on APPROVED it transitions
  to qualifyStep; on DISQUALIFIED it ends. qualifyStep calls QualificationAgent and
  writes recordQualification. Override settings() with stepTimeout(60s) on scoreStep
  and qualifyStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity LeadEntity holding a Lead record with id, companyName
  (Optional<String>), contactEmail (Optional<String>), LeadStatus enum
  {SCORED,APPROVED,DISQUALIFIED,QUALIFIED}, and Optional lifecycle fields
  (scoredAt, score, scoreRationale, scoreConfidence, approvedAt, approvedBy,
  approverNote, rejectedAt, rejectedBy, rejectReason, qualifiedAt,
  qualificationVerdict, qualificationNextSteps). Events: LeadScored, LeadApproved,
  LeadRejected, LeadQualified. Commands: recordScore, approve, reject,
  recordQualification, getLead. emptyState() returns Lead.initial("") with no
  commandContext() reference (Lesson 3).
- 1 View LeadsView with row type Lead, table updater consuming LeadEntity events.
  ONE query: getAllLeads SELECT * AS leads FROM leads_view. No WHERE status filter
  (Akka cannot auto-index enum columns, Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: LeadScoringEndpoint at /api with score-request (starts a
  LeadScoringWorkflow with a fresh UUID), approve, reject, leads list (filter
  client-side from getAllLeads), single lead, SSE stream, and three /api/metadata/*
  endpoints serving the YAML/MD files from src/main/resources/metadata/.
  AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- LeadScoringTasks.java declaring two Task<R> constants: SCORE (resultConformsTo
  LeadScore) and QUALIFY (resultConformsTo QualificationSummary).
- LeadScore(int score, String rationale, String confidence),
  ReviewDecision(String approvedBy, String note),
  QualificationSummary(String verdict, String nextSteps).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9620
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (H1, E1, G1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects; marking jurisdictions, declared frameworks,
  organization fields, incidents, and data.residency as TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: elevator pitch, component inventory, matrix cell,
  integration descriptor, how to run, the 5 tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs: Overview, Architecture, Risk
  Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ScoringAgent -> LeadScore, QualificationAgent -> QualificationSummary; see
  src/main/resources/mock-responses/{scoring-agent,qualification-agent}.json with 4–6
  entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option a):
- scoring-agent.json: 4–6 entries, each { "score": 0–100, "rationale": "...",
  "confidence": "high | medium | low" }.
- qualification-agent.json: 4–6 entries, each { "verdict": "...",
  "nextSteps": "..." }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  LeadScoringTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9620 declared in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes mermaid CSS overrides and theme variables
  (state-label colour, edge-label overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching matches by data-tab/data-panel attribute, never
  NodeList index; no hidden zombie panels in the DOM.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
