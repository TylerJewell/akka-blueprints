# SPEC — user-confirmation-agent

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** User Confirmation Agents.
**One-line pitch:** A user submits a request; `PlannerAgent` proposes an action plan; the workflow pauses at a confirmation task; a human confirms or cancels through the API; on confirmation `ExecutorAgent` executes the plan and returns a summary.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the general domain: a 3-task graph that plans, then waits at an unassigned confirmation task that a person completes through the API, then executes only if confirmed. The governance pattern is an application-level human confirmation gate between the planning and execution phases, and a before-tool-call guardrail that blocks execution unless the request has been explicitly confirmed.

## 3. User-facing flows

1. A client POSTs a request description to `/api/confirmation-request`. The response returns `{ requestId }`. The request appears in the UI in `PENDING_CONFIRMATION` once `PlannerAgent` finishes (typically 5–30 s), with the proposed actions visible for review.
2. The user reads the proposed plan and clicks Confirm. This POSTs to `/api/requests/{requestId}/confirm`. The workflow resumes, `ExecutorAgent` runs, and the execution summary appears with status `EXECUTED`.
3. The user clicks Cancel with a reason. This POSTs to `/api/requests/{requestId}/cancel`. The request moves to terminal `CANCELLED` and the reason is shown. The execute step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| PlannerAgent | AutonomousAgent | Plans an action set for the request; returns `ActionPlan{actions}` | ConfirmationWorkflow | RequestEntity |
| ExecutorAgent | AutonomousAgent | Executes the confirmed plan; returns `ExecutionResult{summary, executedAt}` | ConfirmationWorkflow | RequestEntity |
| ConfirmationWorkflow | Workflow | Orchestrates plan → await confirmation → execute | ConfirmationEndpoint | PlannerAgent, ExecutorAgent, RequestEntity |
| RequestEntity | EventSourcedEntity | Holds the request state and lifecycle events | ConfirmationWorkflow, ConfirmationEndpoint | RequestsView |
| RequestsView | View | CQRS read model of all requests | RequestEntity | ConfirmationEndpoint |
| ConfirmationEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | ConfirmationWorkflow, RequestEntity, RequestsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Request` (RequestEntity state and RequestsView row): `id` (String), `description` (`Optional<String>`), `status` (RequestStatus enum), and lifecycle fields all `Optional<T>`: `plannedAt`, `proposedActions`, `confirmedAt`, `confirmedBy`, `confirmationNote`, `cancelledAt`, `cancelledBy`, `cancelReason`, `executedAt`, `executionSummary`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`RequestStatus` enum: `PENDING_CONFIRMATION`, `CONFIRMED`, `CANCELLED`, `EXECUTED`.

Events: `RequestPlanned`, `RequestConfirmed`, `RequestCancelled`, `RequestExecuted`.

Domain records: `ActionPlan(List<String> actions)`, `ConfirmationDecision(String confirmedBy, String note)`, `ExecutionResult(String summary, String executedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/confirmation-request              -> { requestId }
POST /api/requests/{requestId}/confirm      -> 200 | 404
POST /api/requests/{requestId}/cancel       -> 200 | 404
GET  /api/requests                          -> { requests: [Request, ...] }
GET  /api/requests/{requestId}              -> Request
GET  /api/requests/sse                      -> Server-Sent Events of Request
GET  /api/metadata/eval-matrix              -> text/yaml
GET  /api/metadata/risk-survey              -> text/yaml
GET  /api/metadata/readme                   -> text/markdown
GET  /                                      -> 302 /app/index.html
GET  /app/{*path}                           -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: User Confirmation Agents</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a request description, lists requests live via SSE, and shows Confirm/Cancel buttons on `PENDING_CONFIRMATION` requests that have proposed actions. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `ConfirmationWorkflow` pauses at the await-confirmation task; `/api/requests/{id}/confirm` and `/api/requests/{id}/cancel` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `ExecutorAgent` verifies `RequestEntity.status == CONFIRMED` before the action execution tool runs.

## 9. Agent prompts

- `PlannerAgent` — plans a set of actions for a user request. See `prompts/planner-agent.md`.
- `ExecutorAgent` — executes a confirmed action plan and returns a summary. See `prompts/executor-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Plan a request.** POST a description; within ~30 s a request appears in `PENDING_CONFIRMATION` with non-empty `proposedActions`.
2. **Confirm and execute.** Confirm a `PENDING_CONFIRMATION` request; it reaches `EXECUTED` with a non-null `executionSummary` within ~30 s.
3. **Cancel a request.** Cancel a `PENDING_CONFIRMATION` request with a reason; it moves to terminal `CANCELLED` and the reason shows.
4. **Execution guard.** The execute step is never reached for a request that is not `CONFIRMED`.

---

## 11. Implementation directives

```
Create a sample named user-confirmation-agent demonstrating the human-in-loop-gate ×
general cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-general-user-confirmation-agent.
Java package io.akka.samples.userconfirmationagents.
Akka 3.6.0. HTTP port 9193.

Components to wire (exactly):
- 2 AutonomousAgents: PlannerAgent (plans actions for a request, returns a typed
  ActionPlan{List<String> actions}) and ExecutorAgent (returns a typed
  ExecutionResult{summary, executedAt}). Each declares definition() returning an
  AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow ConfirmationWorkflow with three tasks: planStep -> awaitConfirmationStep
  -> executeStep. planStep calls forAutonomousAgent(PlannerAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordPlan on
  RequestEntity. awaitConfirmationStep polls RequestEntity.getRequest; on
  PENDING_CONFIRMATION it self-schedules a 5-second resume timer; on CONFIRMED it
  transitions to executeStep; on CANCELLED it ends. executeStep calls ExecutorAgent
  and writes recordExecution. Override settings() with stepTimeout(60s) on planStep
  and executeStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity RequestEntity holding a Request record with id, description
  (Optional<String>), RequestStatus enum
  {PENDING_CONFIRMATION,CONFIRMED,CANCELLED,EXECUTED}, and Optional lifecycle fields
  (plannedAt, proposedActions, confirmedAt, confirmedBy, confirmationNote,
  cancelledAt, cancelledBy, cancelReason, executedAt, executionSummary). Events:
  RequestPlanned, RequestConfirmed, RequestCancelled, RequestExecuted. Commands:
  recordPlan, confirm, cancel, recordExecution, getRequest. emptyState() returns
  Request.initial("") with no commandContext() reference (Lesson 3).
- 1 View RequestsView with row type Request, table updater consuming RequestEntity
  events. ONE query: getAllRequests SELECT * AS requests FROM requests_view. No
  WHERE status filter (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side in callers.
- 2 HttpEndpoints: ConfirmationEndpoint at /api with confirmation-request (starts a
  ConfirmationWorkflow with a fresh UUID), confirm, cancel, requests list (filter
  client-side from getAllRequests), single request, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- ConfirmationTasks.java declaring two Task<R> constants: PLAN (resultConformsTo
  ActionPlan) and EXECUTE (resultConformsTo ExecutionResult).
- ActionPlan(List<String> actions), ConfirmationDecision(String confirmedBy,
  String note), ExecutionResult(String summary, String executedAt).
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9193 and akka.javasdk.agent model-provider
  blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
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
  (PlannerAgent -> ActionPlan, ExecutorAgent -> ExecutionResult; see
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
- planner-agent.json: 4–6 entries, each { "actions": ["action1", "action2", ...] }
  with 2–4 plausible action strings per entry.
- executor-agent.json: 4–6 entries, each { "summary": "...", "executedAt":
  "ISO-8601" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  ConfirmationTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9193 declared in application.conf.
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
