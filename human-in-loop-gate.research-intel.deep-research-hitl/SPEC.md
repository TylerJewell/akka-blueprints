# SPEC — deep-research-hitl

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** HITL Deep Research.
**One-line pitch:** A user submits a research query; `PlannerAgent` breaks it into sub-topics; `ResearchAgent` investigates each; `SynthesisAgent` assembles a report draft; the workflow pauses at a human review gate; on approval the report is delivered with a timestamp, or rejection sends it back for revision.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the research-intel domain: a multi-agent pipeline that plans, investigates (fan-out), and synthesises before pausing for a human reviewer who either approves delivery or requests revision. The governance pattern is an application-level human approval gate between synthesis and delivery, an output guardrail before the synthesised draft reaches the reviewer, and a tool-permission guardrail that blocks the deliver step unless the report is approved.

## 3. User-facing flows

1. A client POSTs a query to `/api/research-request`. The response returns `{ reportId }`. The report appears in the UI in `PLANNING`, then progresses to `INVESTIGATING` (one card per sub-topic), then `SYNTHESISING`, then `AWAITING_REVIEW` once `SynthesisAgent` finishes (typically 30–120 s), with the full report body visible.
2. The reviewer reads the synthesised report and clicks Approve. This POSTs to `/api/reports/{reportId}/approve`. The workflow resumes, the report transitions to `DELIVERED`, and a delivery timestamp appears.
3. The reviewer clicks Reject with revision notes. This POSTs to `/api/reports/{reportId}/reject`. The report moves to `NEEDS_REVISION`; the workflow re-enters the synthesis step with the revision notes appended to context, then returns to `AWAITING_REVIEW` for another review cycle.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| PlannerAgent | AutonomousAgent | Decomposes a research query into sub-topics; returns `ResearchPlan{queryId, subTopics}` | ResearchWorkflow | ResearchEntity |
| ResearchAgent | AutonomousAgent | Investigates one sub-topic; returns `SubTopicFindings{subTopic, summary, sources}` | ResearchWorkflow | ResearchEntity |
| SynthesisAgent | AutonomousAgent | Assembles findings into a report draft; returns `ReportDraft{title, body, sourcesUsed}` | ResearchWorkflow | ResearchEntity |
| ResearchWorkflow | Workflow | Orchestrates plan → investigate → synthesise → await review → deliver | ResearchEndpoint | PlannerAgent, ResearchAgent, SynthesisAgent, ResearchEntity |
| ResearchEntity | EventSourcedEntity | Holds the report state and lifecycle events | ResearchWorkflow, ResearchEndpoint | ReportsView |
| ReportsView | View | CQRS read model of all reports | ResearchEntity | ResearchEndpoint |
| ResearchEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | ResearchWorkflow, ResearchEntity, ReportsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Report` (ResearchEntity state and ReportsView row): `id` (String), `query` (`Optional<String>`), `status` (ReportStatus enum), and lifecycle fields all `Optional<T>`: `planCreatedAt`, `subTopics` (`Optional<List<String>>`), `findingsSummaries` (`Optional<List<String>>`), `synthesisStartedAt`, `reportTitle`, `reportBody`, `sourcesUsed` (`Optional<List<String>>`), `awaitingReviewAt`, `approvedAt`, `approvedBy`, `approverNotes`, `rejectedAt`, `rejectedBy`, `revisionNotes`, `deliveredAt`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`ReportStatus` enum: `PLANNING`, `INVESTIGATING`, `SYNTHESISING`, `AWAITING_REVIEW`, `APPROVED`, `NEEDS_REVISION`, `DELIVERED`.

Events: `QueryPlanned`, `InvestigationStarted`, `SynthesisStarted`, `ReportAwaitingReview`, `ReviewApproved`, `ReviewRejected`, `ReportDelivered`.

Domain records: `ResearchPlan(String queryId, List<String> subTopics)`, `SubTopicFindings(String subTopic, String summary, List<String> sources)`, `ReportDraft(String title, String body, List<String> sourcesUsed)`, `ReviewDecision(String reviewedBy, String notes)`, `DeliveredReport(String reportId, String deliveredAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/research-request           -> { reportId }
POST /api/reports/{reportId}/approve -> 200 | 404
POST /api/reports/{reportId}/reject  -> 200 | 404
GET  /api/reports                    -> { reports: [Report, ...] }
GET  /api/reports/{reportId}         -> Report
GET  /api/reports/sse                -> Server-Sent Events of Report
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: HITL Deep Research</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a query, lists reports live via SSE, and shows Approve/Reject buttons on `AWAITING_REVIEW` reports that have a report body. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `ResearchWorkflow` pauses at the await-review task; `/api/reports/{id}/approve` and `/api/reports/{id}/reject` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `SynthesisAgent` (acting as the final delivery agent) verifies `ResearchEntity.status == APPROVED` before the simulated deliver tool runs.
- **G2 — guardrail · before-agent-response.** A guardrail on `SynthesisAgent` checks report length and source-citation presence before the draft is persisted for review.

## 9. Agent prompts

- `PlannerAgent` — decomposes a research query into focused sub-topics. See `prompts/planner-agent.md`.
- `ResearchAgent` — investigates one sub-topic and returns findings with cited sources. See `prompts/research-agent.md`.
- `SynthesisAgent` — assembles per-sub-topic findings into a coherent, cited report draft. See `prompts/synthesis-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Plan a query.** POST a query; within ~15 s the report appears in `PLANNING` then advances to `INVESTIGATING` with a non-empty `subTopics` list.
2. **Await synthesis.** Within ~60–120 s the report moves to `AWAITING_REVIEW` with non-empty `reportBody` and at least one source in `sourcesUsed`.
3. **Approve and deliver.** Approve an `AWAITING_REVIEW` report; it reaches `DELIVERED` with a non-null `deliveredAt` within ~5 s.
4. **Reject and revise.** Reject an `AWAITING_REVIEW` report with revision notes; it moves to `NEEDS_REVISION` then back to `AWAITING_REVIEW` after re-synthesis.
5. **Deliver guard.** The deliver step never runs for a report that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named deep-research-hitl demonstrating the human-in-loop-gate ×
research-intel cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-research-intel-deep-research-hitl.
Java package io.akka.samples.deepsearch. Akka 3.6.0. HTTP port 9974.

Components to wire (exactly):
- 3 AutonomousAgents:
  PlannerAgent (decomposes a research query, returns a typed
  ResearchPlan{queryId, subTopics}) and ResearchAgent (investigates one
  sub-topic, returns a typed SubTopicFindings{subTopic, summary, sources}) and
  SynthesisAgent (assembles findings into a report, returns a typed
  ReportDraft{title, body, sourcesUsed}). Each declares definition() returning an
  AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). All extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow ResearchWorkflow with four tasks: planStep -> investigateStep ->
  synthesiseStep -> awaitReviewStep -> deliverStep. planStep calls
  forAutonomousAgent(PlannerAgent.class,...).runSingleTask(...), writes
  recordPlan on ResearchEntity. investigateStep fans out one ResearchAgent
  task per sub-topic in sequence and writes recordFindings after each returns.
  synthesiseStep calls SynthesisAgent.runSingleTask(SYNTHESISE), writes
  recordDraft on ResearchEntity. awaitReviewStep polls ResearchEntity.getReport;
  on AWAITING_REVIEW it self-schedules a 5-second resume timer; on APPROVED it
  transitions to deliverStep; on NEEDS_REVISION it loops back to synthesiseStep
  with revision notes. deliverStep writes recordDelivery on ResearchEntity.
  Override settings() with stepTimeout(120s) on planStep, each investigateStep
  iteration, and synthesiseStep; stepTimeout(60s) on deliverStep.
  WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity ResearchEntity holding a Report record with id, query
  (Optional<String>), ReportStatus enum
  {PLANNING,INVESTIGATING,SYNTHESISING,AWAITING_REVIEW,APPROVED,NEEDS_REVISION,DELIVERED},
  and Optional lifecycle fields (planCreatedAt, subTopics as Optional<List<String>>,
  findingsSummaries as Optional<List<String>>, synthesisStartedAt, reportTitle,
  reportBody, sourcesUsed as Optional<List<String>>, awaitingReviewAt, approvedAt,
  approvedBy, approverNotes, rejectedAt, rejectedBy, revisionNotes, deliveredAt).
  Events: QueryPlanned, InvestigationStarted, SynthesisStarted,
  ReportAwaitingReview, ReviewApproved, ReviewRejected, ReportDelivered.
  Commands: recordPlan, recordFindings, recordDraft, approve, reject,
  recordDelivery, getReport. emptyState() returns Report.initial("") with no
  commandContext() reference (Lesson 3).
- 1 View ReportsView with row type Report, table updater consuming ResearchEntity
  events. ONE query: getAllReports SELECT * AS reports FROM reports_view. No WHERE
  status filter (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side in callers.
- 2 HttpEndpoints: ResearchEndpoint at /api with research-request (starts a
  ResearchWorkflow with a fresh UUID), approve, reject, reports list (filter
  client-side from getAllReports), single report, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- ResearchTasks.java declaring three Task<R> constants: PLAN (resultConformsTo
  ResearchPlan), INVESTIGATE (resultConformsTo SubTopicFindings), SYNTHESISE
  (resultConformsTo ReportDraft).
- ResearchPlan(String queryId, List<String> subTopics),
  SubTopicFindings(String subTopic, String summary, List<String> sources),
  ReportDraft(String title, String body, List<String> sourcesUsed),
  ReviewDecision(String reviewedBy, String notes),
  DeliveredReport(String reportId, String deliveredAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9974
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (H1, G1, G2) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions, declared
  frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: elevator pitch, component inventory, matrix cell,
  integration descriptor, how to run, the 5 tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown (marked) and YAML (js-yaml) are acceptable. Five tabs: Overview,
  Architecture, Risk Survey, Eval Matrix, App UI. Match the governance.html visual
  style (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (PlannerAgent -> ResearchPlan, ResearchAgent -> SubTopicFindings,
  SynthesisAgent -> ReportDraft; see
  src/main/resources/mock-responses/{planner-agent,research-agent,synthesis-agent}.json
  with 4–6 entries each). Sets model-provider = mock.
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
- planner-agent.json: 4–6 entries, each { "queryId": "uuid", "subTopics": ["...",
  "...", "..."] } with 3 sub-topics each.
- research-agent.json: 4–6 entries, each { "subTopic": "...", "summary": "2–3
  paragraphs of plausible research findings", "sources": ["url1", "url2"] }.
- synthesis-agent.json: 4–6 entries, each { "title": "...", "body": "4–6
  paragraphs of synthesised prose with inline citations", "sourcesUsed":
  ["url1", "url2", "url3"] }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(120s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  ResearchTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9974 declared in application.conf.
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

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
