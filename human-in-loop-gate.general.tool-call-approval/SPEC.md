# SPEC — tool-call-approval

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** HITL Tool Approval.
**One-line pitch:** A user submits a goal; `PlannerAgent` selects a tool and its parameters; the workflow pauses at an approval task; a human approves, edits parameters, or rejects through the API; on approval `ExecutorAgent` runs the tool and returns the result.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the general domain: a 3-task graph that plans a tool call, waits at an unassigned approval task that a person completes through the API, then executes only if approved. The governance pattern is an application-level human approval gate between the plan and execute phases, and a before-tool-call guardrail that blocks the execute step unless the request is in `APPROVED` status.

## 3. User-facing flows

1. A client POSTs a goal to `/api/tool-requests`. The response returns `{ requestId }`. The request appears in the UI in `PENDING_APPROVAL` once `PlannerAgent` finishes (typically 5–30 s), with the planned tool name, parameters, and rationale visible.
2. The operator reads the plan and clicks Approve. This POSTs to `/api/tool-requests/{requestId}/approve`. The workflow resumes, `ExecutorAgent` runs, and the output appears with status `EXECUTED`.
3. The operator edits the parameters before approving. This POSTs to `/api/tool-requests/{requestId}/approve` with an updated `parameters` map. The workflow resumes using the edited parameters.
4. The operator clicks Reject with a reason. This POSTs to `/api/tool-requests/{requestId}/reject`. The request moves to terminal `REJECTED` and the reason is shown. The execute step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| PlannerAgent | AutonomousAgent | Selects a tool and builds parameters for a given goal; returns `ToolCallPlan{toolName, parameters, rationale}` | ApprovalWorkflow | ToolRequestEntity |
| ExecutorAgent | AutonomousAgent | Executes an approved tool call; returns `ToolCallResult{output, executedAt}` | ApprovalWorkflow | ToolRequestEntity |
| ApprovalWorkflow | Workflow | Orchestrates plan → await approval → execute | ApprovalEndpoint | PlannerAgent, ExecutorAgent, ToolRequestEntity |
| ToolRequestEntity | EventSourcedEntity | Holds the tool-request state and lifecycle events | ApprovalWorkflow, ApprovalEndpoint | ToolRequestsView |
| ToolRequestsView | View | CQRS read model of all tool requests | ToolRequestEntity | ApprovalEndpoint |
| ApprovalEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | ApprovalWorkflow, ToolRequestEntity, ToolRequestsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`ToolRequest` (ToolRequestEntity state and ToolRequestsView row): `id` (String), `goal` (`Optional<String>`), `status` (ToolRequestStatus enum), and lifecycle fields all `Optional<T>`: `plannedAt`, `toolName`, `parameters`, `rationale`, `approvedAt`, `approvedBy`, `approverNote`, `editedParameters`, `rejectedAt`, `rejectedBy`, `rejectReason`, `executedAt`, `executionOutput`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`ToolRequestStatus` enum: `PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `EXECUTED`.

Events: `ToolCallPlanned`, `ToolCallApproved`, `ToolCallRejected`, `ToolCallExecuted`.

Domain records: `ToolCallPlan(String toolName, String parameters, String rationale)`, `ApprovalDecision(String approvedBy, String approverNote, Optional<String> editedParameters)`, `ToolCallResult(String output, String executedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/tool-requests                          -> { requestId }
POST /api/tool-requests/{requestId}/approve      -> 200 | 404
POST /api/tool-requests/{requestId}/reject       -> 200 | 404
GET  /api/tool-requests                          -> { requests: [ToolRequest, ...] }
GET  /api/tool-requests/{requestId}              -> ToolRequest
GET  /api/tool-requests/sse                      -> Server-Sent Events of ToolRequest
GET  /api/metadata/eval-matrix                   -> text/yaml
GET  /api/metadata/risk-survey                   -> text/yaml
GET  /api/metadata/readme                        -> text/markdown
GET  /                                           -> 302 /app/index.html
GET  /app/{*path}                                -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: HITL Tool Approval</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a goal, lists requests live via SSE, and shows Approve/Edit+Approve/Reject buttons on `PENDING_APPROVAL` requests that have a plan. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `ApprovalWorkflow` pauses at the await-approval task; `/api/tool-requests/{id}/approve` and `/api/tool-requests/{id}/reject` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `ExecutorAgent` verifies `ToolRequestEntity.status == APPROVED` before the simulated tool runs.

## 9. Agent prompts

- `PlannerAgent` — selects a tool and builds call parameters for a goal. See `prompts/planner-agent.md`.
- `ExecutorAgent` — executes an approved tool call and returns its output. See `prompts/executor-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Plan a tool call.** POST a goal; within ~30 s a request appears in `PENDING_APPROVAL` with non-empty `toolName` and `parameters`.
2. **Approve and execute.** Approve a `PENDING_APPROVAL` request; it reaches `EXECUTED` with a non-null `executionOutput` within ~30 s.
3. **Edit parameters and approve.** Approve with an `editedParameters` override; the executor uses the edited parameters, not the original plan.
4. **Reject a plan.** Reject a `PENDING_APPROVAL` request with a reason; it moves to terminal `REJECTED` and the reason shows.
5. **Execute guard.** The execute step is never reached for a request that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named tool-call-approval demonstrating the human-in-loop-gate ×
general cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-general-tool-call-approval.
Java package io.akka.samples.humaninthelooptoolapproval.
Akka 3.6.0. HTTP port 9225.

Components to wire (exactly):
- 2 AutonomousAgents: PlannerAgent (selects a tool for the given goal, returns a
  typed ToolCallPlan{toolName, parameters, rationale}) and ExecutorAgent (simulates
  running the approved tool, returns a typed ToolCallResult{output, executedAt}).
  Each declares definition() returning an AgentDefinition with .instructions(...)
  loaded from prompts and .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)).
  Both extend akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow ApprovalWorkflow with three tasks: planStep -> awaitApprovalStep ->
  executeStep. planStep calls forAutonomousAgent(PlannerAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordPlan on
  ToolRequestEntity. awaitApprovalStep polls ToolRequestEntity.getRequest; on
  PENDING_APPROVAL it self-schedules a 5-second resume timer; on APPROVED it
  transitions to executeStep; on REJECTED it ends. executeStep reads the approved
  (or edited) parameters from the entity, calls ExecutorAgent, and writes
  recordExecution. Override settings() with stepTimeout(60s) on planStep and
  executeStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity ToolRequestEntity holding a ToolRequest record with id, goal
  (Optional<String>), ToolRequestStatus enum
  {PENDING_APPROVAL,APPROVED,REJECTED,EXECUTED}, and Optional lifecycle fields
  (plannedAt, toolName, parameters, rationale, approvedAt, approvedBy, approverNote,
  editedParameters, rejectedAt, rejectedBy, rejectReason, executedAt,
  executionOutput). Events: ToolCallPlanned, ToolCallApproved, ToolCallRejected,
  ToolCallExecuted. Commands: recordPlan, approve, reject, recordExecution,
  getRequest. emptyState() returns ToolRequest.initial("") with no commandContext()
  reference (Lesson 3).
- 1 View ToolRequestsView with row type ToolRequest, table updater consuming
  ToolRequestEntity events. ONE query: getAllRequests SELECT * AS requests FROM
  tool_requests_view. No WHERE status filter (Akka cannot auto-index enum columns,
  Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: ApprovalEndpoint at /api with tool-requests (starts an
  ApprovalWorkflow with a fresh UUID), approve (accepts optional editedParameters),
  reject, requests list (filter client-side from getAllRequests), single request,
  SSE stream, and three /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- ApprovalTasks.java declaring two Task<R> constants: PLAN (resultConformsTo
  ToolCallPlan) and EXECUTE (resultConformsTo ToolCallResult).
- ToolCallPlan(String toolName, String parameters, String rationale),
  ApprovalDecision(String approvedBy, String approverNote,
  Optional<String> editedParameters), ToolCallResult(String output, String executedAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9225
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
  (PlannerAgent -> ToolCallPlan, ExecutorAgent -> ToolCallResult; see
  src/main/resources/mock-responses/{planner-agent,executor-agent}.json with 4–6
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
- planner-agent.json: 4–6 entries, each { "toolName": "...", "parameters": "...",
  "rationale": "1–2 sentences of reasoning" }.
- executor-agent.json: 4–6 entries, each { "output":
  "Simulated result: ...", "executedAt": "ISO-8601" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  ApprovalTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9225 declared in application.conf.
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
