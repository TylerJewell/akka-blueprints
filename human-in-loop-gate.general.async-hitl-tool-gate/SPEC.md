# Async HITL Tool Gate

## 1. System name + one-line pitch

**Async HITL Tool Gate.** A user submits a request; an agent plans a tool call; the workflow pauses for human approval; on approval a second agent runs the tool and returns its result.

## 2. What this blueprint demonstrates

The **human-in-loop-gate** coordination pattern in the **general** domain: a durable async workflow that pauses before a side-effecting tool call until a human approves the agent's planned action. The governance pattern is a blocking human approval gate (`hitl · application`) complemented by a `guardrail · before-tool-call` policy check that refuses tool execution unless the action's status is `APPROVED`. Built with first-party Akka primitives — no external queue, no manual coordination protocol.

## 3. User-facing flows

1. A client POSTs a request (`{ "request": "..." }`) to `/api/requests`. The response carries the `actionId`. The UI streams `/api/actions/sse` and the action appears in `PLANNED` state with the agent's proposed tool, arguments, and rationale (typically 5–30 s after submit).
2. The reviewer reads the planned action and clicks Approve. The UI POSTs `/api/actions/{id}/approve`. The workflow resumes, the `before-tool-call` guardrail confirms `APPROVED`, `ExecutorAgent` runs the tool, and the result appears with status `EXECUTED`.
3. The reviewer clicks Reject with a reason. The workflow ends in a terminal `REJECTED` state; the UI shows the reason.
4. A `PLANNED` action left untouched for more than 2 minutes is marked `ESCALATED` by `StuckActionMonitor`; the Approve/Reject buttons disappear.
5. Without any UI interaction, `RequestSimulator` drips canned requests from a JSONL file; `RequestConsumer` starts one workflow per request.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ActionAgent` | AutonomousAgent | Plans a tool call; returns typed `ActionPlan` | `ApprovalWorkflow.planStep` | `ActionEntity` |
| `ExecutorAgent` | AutonomousAgent | Runs the approved tool; returns typed `ToolResult` | `ApprovalWorkflow.executeStep` | `ActionEntity` |
| `ApprovalTasks` | task definitions | `Task<R>` constants `PLAN`, `EXECUTE` | — | the two agents |
| `ActionEntity` | EventSourcedEntity | Durable per-action state and lifecycle | workflow, endpoint, monitor | `ActionsView` |
| `InboundRequestQueue` | EventSourcedEntity | Single-instance inbound request log | `RequestSimulator`, endpoint | `RequestConsumer` |
| `ActionsView` | View | CQRS read model; row type `Action` | `ActionEntity` events | endpoint, monitor |
| `ApprovalWorkflow` | Workflow | `planStep` → `awaitApprovalStep` → `executeStep` | `RequestConsumer` | agents, `ActionEntity` |
| `RequestConsumer` | Consumer | Starts one workflow per queued request | `InboundRequestQueue` | `ApprovalWorkflow` |
| `RequestSimulator` | TimedAction | Drips canned requests every 30 s | JSONL file | `InboundRequestQueue` |
| `StuckActionMonitor` | TimedAction | Escalates `PLANNED` actions older than 2 min | `ActionsView` | `ActionEntity` |
| `ActionEndpoint` | HttpEndpoint | `/api/*` REST + SSE + metadata | UI, clients | workflow, entity, view |
| `AppEndpoint` | HttpEndpoint | Serves `/` redirect and `/app/*` static | browser | static-resources |
| `Bootstrap` | service-setup | Schedules TimedActions on startup | — | TimedActions |

## 5. Data model

See `reference/data-model.md`. Authoritative records:

```java
public record Action(
  String id,
  Optional<String>  request,
  ActionStatus      status,
  Optional<Instant> plannedAt,
  Optional<String>  toolName,
  Optional<String>  toolArguments,
  Optional<String>  rationale,
  Optional<Instant> approvedAt,
  Optional<String>  approvedBy,
  Optional<String>  approverComment,
  Optional<Instant> rejectedAt,
  Optional<String>  rejectedBy,
  Optional<String>  rejectReason,
  Optional<Instant> executedAt,
  Optional<String>  toolOutput,
  Optional<Instant> escalatedAt
) {
  public static Action initial(String id, String request) { /* all Optional.empty() */ }
}

enum ActionStatus { CREATED, PLANNED, APPROVED, REJECTED, EXECUTED, ESCALATED }

record ActionPlan(String toolName, String toolArguments, String rationale) {}
record ToolResult(String toolOutput, String executedAt) {}
record ApprovalDecision(String approvedBy, String comment) {}
```

Events (sealed `ActionEvent`): `ActionPlanned`, `ActionApproved`, `ActionRejected`, `ActionExecuted`, `ActionEscalated`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

## 6. API contract

Full surface in `reference/api-contract.md`. Top-level:

```
POST /api/requests                  -> { actionId }
POST /api/actions/{id}/approve      -> 200 | 404
POST /api/actions/{id}/reject       -> 200 | 404
GET  /api/actions ?status=...       -> { actions: [Action, ...] }
GET  /api/actions/{id}              -> Action
GET  /api/actions/sse               -> SSE of Action
GET  /api/metadata/eval-matrix      -> text/yaml
GET  /api/metadata/risk-survey      -> text/yaml
GET  /api/metadata/readme           -> text/markdown
GET  /                              -> 302 /app/index.html
GET  /app/{*path}                   -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/`, no npm. Browser title `<title>Akka Sample: Async HITL Tool Gate</title>`. Five tabs: **Overview / Architecture / Risk Survey / Eval Matrix / App UI**. See `reference/ui-mockup.md`.

- Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index; no zombie panels (Lesson 26).
- Architecture tab renders the mermaid diagrams with the state-label CSS overrides and theme variables from Lesson 24.
- App UI tab: submit a request, live SSE list of actions, per-action Approve/Reject buttons when status is `PLANNED`.

Visual style anchor: `specs/vision/governance.html` (dark / yellow accent / Instrument Sans / dot-grid).

## 8. Governance

Controls in `eval-matrix.yaml`; pre-filled deployer answers in `risk-survey.yaml`. Two mechanisms the generated system must wire:

- **H1 (`hitl · application`)** — `ApprovalWorkflow` pauses in `awaitApprovalStep` until `POST /api/actions/{id}/approve` or `/reject` transitions it.
- **G1 (`guardrail · before-tool-call`)** — a before-tool-call guardrail on `ExecutorAgent` refuses the tool unless `ActionEntity.status == APPROVED`.

## 9. Agent prompts

- `prompts/action-agent.md` — system prompt for `ActionAgent`: given a request, propose one tool call with arguments and a short rationale.
- `prompts/executor-agent.md` — system prompt for `ExecutorAgent`: run the approved tool and report its result.

## 10. Acceptance

See `reference/user-journeys.md`. The journeys that mean "generated correctly":

1. **J1 — Plan.** POST a request; within ~30 s the action reaches `PLANNED` with non-empty `toolName` and `rationale`.
2. **J2 — Approve and execute.** Approve a `PLANNED` action; within ~30 s it reaches `EXECUTED` with non-empty `toolOutput`.
3. **J3 — Reject.** Reject a `PLANNED` action; it reaches terminal `REJECTED` with the reason recorded.
4. **J4 — Escalate.** A `PLANNED` action untouched for > 2 min reaches `ESCALATED` automatically.

## 11. Implementation directives

```
Create a sample named async-hitl-tool-gate demonstrating the
human-in-loop-gate × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact async-hitl-tool-gate. Java package
io.akka.samples.asynchitltoolgate. Akka 3.6.0. HTTP port 9237.
Matrix cell: coordination_pattern=human-in-loop-gate, domain=general.

Components to wire (exactly):
- 2 AutonomousAgents: ActionAgent (returns typed ActionPlan{toolName,
  toolArguments, rationale}) and ExecutorAgent (returns typed
  ToolResult{toolOutput, executedAt}). Each declares definition() with
  capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). ExecutorAgent
  carries a before-tool-call guardrail that verifies ActionEntity status is
  APPROVED before the simulated tool runs.
- 1 Workflow ApprovalWorkflow with steps planStep -> awaitApprovalStep ->
  executeStep. planStep calls forAutonomousAgent(ActionAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), then
  ActionEntity.recordPlan. awaitApprovalStep polls ActionEntity.getAction; on
  PLANNED self-schedules a 5-second resume timer; on APPROVED transitions to
  executeStep; on REJECTED or ESCALATED ends. executeStep calls
  forAutonomousAgent(ExecutorAgent.class,...) then ActionEntity.recordExecution.
  Override settings() with stepTimeout(60s) on planStep and executeStep,
  stepTimeout(10s) on awaitApprovalStep. WorkflowSettings is nested in Workflow;
  no import.
- 1 EventSourcedEntity ActionEntity holding an Action record with id, request
  (Optional<String>), ActionStatus enum, and the Optional lifecycle fields
  (plannedAt, toolName, toolArguments, rationale, approvedAt, approvedBy,
  approverComment, rejectedAt, rejectedBy, rejectReason, executedAt, toolOutput,
  escalatedAt). Events: ActionPlanned, ActionApproved, ActionRejected,
  ActionExecuted, ActionEscalated. Commands: recordPlan, approve, reject,
  recordExecution, markEscalated, getAction. emptyState() returns
  Action.initial("", null) with no commandContext() reference.
- 1 EventSourcedEntity InboundRequestQueue with a single command
  enqueueRequest(request) emitting InboundRequestQueued.
- 1 View ActionsView with row type Action, table updater consuming ActionEntity
  events. ONE query: getAllActions SELECT * AS actions FROM actions_view. No
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side.
- 1 Consumer RequestConsumer subscribed to InboundRequestQueue events; on each
  event starts an ApprovalWorkflow with a fresh UUID.
- 2 TimedActions: RequestSimulator (every 30s, reads next line from
  src/main/resources/sample-events/requests.jsonl and calls
  InboundRequestQueue.enqueueRequest); StuckActionMonitor (every 30s, queries
  ActionsView.getAllActions, filters PLANNED with plannedAt > 2 min ago, calls
  ActionEntity.markEscalated).
- 2 HttpEndpoints: ActionEndpoint at /api with requests, approve, reject,
  actions list (filter client-side from getAllActions), single action, SSE
  stream, and three /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.
- 1 service-setup Bootstrap scheduling the two TimedActions on startup; fails
  fast if the configured model-provider key reference does not resolve.

Companion files:
- ApprovalTasks.java declaring two Task<R> constants: PLAN (resultConformsTo
  ActionPlan), EXECUTE (resultConformsTo ToolResult).
- ActionPlan(String toolName, String toolArguments, String rationale),
  ToolResult(String toolOutput, String executedAt), ApprovalDecision(String
  approvedBy, String comment).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9237 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/requests.jsonl with 8 canned request lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (G1, H1) and matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking deployer fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): elevator pitch, component
  inventory, how to run, the tabs, license. No governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS; runtime CDN imports for
  markdown and YAML are acceptable. Five tabs (Overview, Architecture, Risk
  Survey, Eval Matrix, App UI). Match governance.html visual style.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (ActionAgent -> ActionPlan, ExecutorAgent -> ToolResult; see
  src/main/resources/mock-responses/{action-agent,executor-agent}.json with 4-6
  entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider per-agent shapes:
- ActionAgent -> ActionPlan{toolName: one of [send_email, create_ticket,
  post_message, schedule_meeting], toolArguments: short JSON-ish string,
  rationale: one sentence}.
- ExecutorAgent -> ToolResult{toolOutput: short confirmation string,
  executedAt: ISO-8601 timestamp}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent extends akka.javasdk.agent.autonomous.AutonomousAgent;
  never silently downgraded to Agent.
- Lesson 4: override workflow step timeouts (agent steps 60s).
- Lesson 6: Optional<T> for every nullable field on the Action row record.
- Lesson 7: ApprovalTasks.java companion required for the AutonomousAgents.
- Lesson 8: verify model names current before locking them in.
- Lesson 9: run command is "/akka:build"; never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9237 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the content column with no horizontal scroll.
- Lesson 13: descriptive integration label ("Runs out of the box"), never T1-T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-diagram label CSS overrides + theme variables.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: attribute-based tab switching; delete dead panels, never display:none.
```

## 12. Post-scaffolding workflow

When `/akka:specify` finishes scaffolding the SDD specification, **do not stop and wait for the user**. Continue immediately:

1. `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and `PLAN.md`.
2. `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. `/akka:implement` — execute every task in sequence. On a task failure, capture it, continue with the next task, summarise all failures at the end.
4. `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service running, output just the listening URL and a one-line summary of any step-3 failures. **Stop earlier only** for a hard error you cannot work around without asking: an unresolved API-key reference (offer the three valid env vars or the mock provider), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
