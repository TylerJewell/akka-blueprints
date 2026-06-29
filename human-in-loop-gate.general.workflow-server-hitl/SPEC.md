# SPEC — workflow-server-hitl

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** WorkflowServer (REST + Streaming + HITL).
**One-line pitch:** A client submits a job request via REST; `AnalysisAgent` evaluates it and streams progress events; the workflow pauses at an approval gate; a human approves or rejects through the API; on approval `CompletionAgent` finalizes the job and streams the result.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the general domain: a 3-task graph that analyzes a job, then waits at an unassigned approval task that a person completes through the REST API, then runs completion only if approved. Progress events are streamed back to the caller via Server-Sent Events throughout. The governance pattern is an application-level human approval gate between the analysis and completion phases, and a tool-permission guardrail that blocks the completion step unless the job is approved.

## 3. User-facing flows

1. A client POSTs a job request to `/api/jobs`. The response returns `{ jobId }`. The job appears in the UI in `ANALYZED` once `AnalysisAgent` finishes (typically 5–30 s), with the findings and risk level visible. Progress events stream over SSE during analysis.
2. The reviewer reads the analysis and clicks Approve. This POSTs to `/api/jobs/{jobId}/approve`. The workflow resumes, `CompletionAgent` runs, and the job output appears with status `COMPLETED`.
3. The reviewer clicks Reject with a reason. This POSTs to `/api/jobs/{jobId}/reject`. The job moves to terminal `REJECTED` and the reason is shown. The completion step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| AnalysisAgent | AutonomousAgent | Evaluates a job request; returns `AnalysisSummary{findings, riskLevel}` | JobWorkflow | JobEntity |
| CompletionAgent | AutonomousAgent | Finalizes an approved job; returns `JobResult{output, completedAt}` | JobWorkflow | JobEntity |
| JobWorkflow | Workflow | Orchestrates analyze → await approval → complete | WorkflowEndpoint | AnalysisAgent, CompletionAgent, JobEntity |
| JobEntity | EventSourcedEntity | Holds the job state and lifecycle events | JobWorkflow, WorkflowEndpoint | JobsView |
| JobsView | View | CQRS read model of all jobs | JobEntity | WorkflowEndpoint |
| WorkflowEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | JobWorkflow, JobEntity, JobsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Job` (JobEntity state and JobsView row): `id` (String), `requestPayload` (`Optional<String>`), `status` (JobStatus enum), and lifecycle fields all `Optional<T>`: `analyzedAt`, `findings`, `riskLevel`, `approvedAt`, `approvedBy`, `approverNote`, `rejectedAt`, `rejectedBy`, `rejectReason`, `completedAt`, `output`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`JobStatus` enum: `ANALYZED`, `APPROVED`, `REJECTED`, `COMPLETED`.

Events: `JobAnalyzed`, `JobApproved`, `JobRejected`, `JobCompleted`.

Domain records: `AnalysisSummary(String findings, String riskLevel)`, `ApprovalDecision(String approvedBy, String note)`, `JobResult(String output, String completedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/jobs                         -> { jobId }
POST /api/jobs/{jobId}/approve         -> 200 | 404
POST /api/jobs/{jobId}/reject          -> 200 | 404
GET  /api/jobs                         -> { jobs: [Job, ...] }
GET  /api/jobs/{jobId}                 -> Job
GET  /api/jobs/sse                     -> Server-Sent Events of Job
GET  /api/metadata/eval-matrix         -> text/yaml
GET  /api/metadata/risk-survey         -> text/yaml
GET  /api/metadata/readme              -> text/markdown
GET  /                                 -> 302 /app/index.html
GET  /app/{*path}                      -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: WorkflowServer (REST + Streaming + HITL)</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a job request, lists jobs live via SSE, and shows Approve/Reject buttons on `ANALYZED` jobs that have findings. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `JobWorkflow` pauses at the await-approval task; `/api/jobs/{id}/approve` and `/api/jobs/{id}/reject` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `CompletionAgent` verifies `JobEntity.status == APPROVED` before the simulated completion tool runs.

## 9. Agent prompts

- `AnalysisAgent` — evaluates a job request and returns findings and a risk level. See `prompts/analysis-agent.md`.
- `CompletionAgent` — finalizes an approved job and returns the output. See `prompts/completion-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Analyze a job.** POST a job request; within ~30 s a job appears in `ANALYZED` with non-empty `findings` and a `riskLevel`.
2. **Approve and complete.** Approve an `ANALYZED` job; it reaches `COMPLETED` with a non-null `output` within ~30 s.
3. **Reject a job.** Reject an `ANALYZED` job with a reason; it moves to terminal `REJECTED` and the reason shows.
4. **Completion guard.** The completion step is never reached for a job that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named workflow-server-hitl demonstrating the human-in-loop-gate ×
general cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-general-workflow-server-hitl.
Java package io.akka.samples.workflowserverreststreaminghitl.
Akka 3.6.0. HTTP port 9269.

Components to wire (exactly):
- 2 AutonomousAgents: AnalysisAgent (evaluates a job request, returns a typed
  AnalysisSummary{findings,riskLevel}) and CompletionAgent (finalizes an approved
  job, returns a typed JobResult{output,completedAt}). Each declares definition()
  returning an AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow JobWorkflow with three tasks: analyzeStep -> awaitApprovalStep
  -> completeStep. analyzeStep calls forAutonomousAgent(AnalysisAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordAnalysis on
  JobEntity. awaitApprovalStep polls JobEntity.getJob; on ANALYZED it
  self-schedules a 5-second resume timer; on APPROVED it transitions to
  completeStep; on REJECTED it ends. completeStep calls CompletionAgent and writes
  recordCompletion. Override settings() with stepTimeout(60s) on analyzeStep and
  completeStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity JobEntity holding a Job record with id, requestPayload
  (Optional<String>), JobStatus enum {ANALYZED,APPROVED,REJECTED,COMPLETED}, and
  Optional lifecycle fields (analyzedAt, findings, riskLevel, approvedAt,
  approvedBy, approverNote, rejectedAt, rejectedBy, rejectReason, completedAt,
  output). Events: JobAnalyzed, JobApproved, JobRejected, JobCompleted.
  Commands: recordAnalysis, approve, reject, recordCompletion, getJob. emptyState()
  returns Job.initial("") with no commandContext() reference (Lesson 3).
- 1 View JobsView with row type Job, table updater consuming JobEntity events.
  ONE query: getAllJobs SELECT * AS jobs FROM jobs_view. No WHERE status filter
  (Akka cannot auto-index enum columns, Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: WorkflowEndpoint at /api with job submission (starts a
  JobWorkflow with a fresh UUID), approve, reject, jobs list (filter
  client-side from getAllJobs), single job, SSE stream, and three /api/metadata/*
  endpoints serving the YAML/MD files from src/main/resources/metadata/. AppEndpoint
  serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- JobTasks.java declaring two Task<R> constants: ANALYZE (resultConformsTo
  AnalysisSummary) and COMPLETE (resultConformsTo JobResult).
- AnalysisSummary(String findings, String riskLevel), ApprovalDecision(String approvedBy,
  String note), JobResult(String output, String completedAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9269
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (H1, G1) and a
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
  (AnalysisAgent -> AnalysisSummary, CompletionAgent -> JobResult; see
  src/main/resources/mock-responses/{analysis-agent,completion-agent}.json with 4–6
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
- analysis-agent.json: 4–6 entries, each { "findings": "...", "riskLevel": "LOW|MEDIUM|HIGH" }.
- completion-agent.json: 4–6 entries, each { "output":
  "string describing the completed job result", "completedAt": "ISO-8601" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  JobTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9269 declared in application.conf.
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
