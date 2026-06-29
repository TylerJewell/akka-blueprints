# SPEC — computer-use-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ComputerUseAgent.
**One-line pitch:** A user submits a plain-language task; one AI agent executes it step by step on a desktop or web environment by issuing tool calls (screenshot, click, type, scroll, key-press); every action passes through a guardrail before execution, an operator halt can stop the agent at any moment, and high-impact actions pause for the user's explicit confirmation.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `ComputerUseAgent` (AutonomousAgent) drives the entire execution loop; the surrounding components govern what it is allowed to do before each step fires. Three governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs before every action the agent proposes. It evaluates the action type, target selector, and payload against a policy ruleset. Destructive actions (file deletion, account closure, data export to unknown destinations) and out-of-scope actions (navigating to domains not on the allow-list) are rejected before they reach the environment. The rejection is returned to the agent so it can choose an alternative path within its iteration budget.
- An **operator halt** gives the operator (or a monitoring system) a direct stop endpoint. Posting to `/api/tasks/{id}/halt` writes a `TaskHalted` event immediately; the workflow's next step check sees the halt flag and terminates the loop without executing any further actions. The agent is not consulted — the halt is unconditional.
- A **human-in-the-loop confirmation gate** runs for a configurable set of high-impact action types (form submission, file write, account modification). A `ConfirmationPending` event is written to the entity; the workflow pauses in `awaitConfirmationStep`. The user approves or rejects via the UI (or `POST /api/tasks/{id}/confirm`). On approval the action executes; on rejection the agent receives a structured `USER_REJECTED` signal and may attempt a recovery path within its remaining iteration budget.

The blueprint shows that a computer-use agent operating autonomously over a live environment requires pre-execution controls on every tool call — not just a post-hoc audit.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **Task template** from a dropdown (three seeded templates: fill-web-form, navigate-and-extract, bulk-rename-files) or types a free-form task description.
2. The user optionally uploads a **starting screenshot** to give the agent its initial view of the environment. If none is provided, the system uses a seeded placeholder screenshot.
3. The user clicks **Run task**. The UI POSTs to `/api/tasks` and receives a `taskId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s the workflow starts and the card transitions to `RUNNING`.
5. As the agent executes actions, each `ActionExecuted` event lands on the entity. The right pane shows a running action log: action type, target, status (ALLOWED / BLOCKED / CONFIRMED / REJECTED), timestamp.
6. If the agent proposes a high-impact action, the card transitions to `AWAITING_CONFIRMATION`. The UI displays the pending action and two buttons: **Approve** and **Reject**.
7. The user approves or rejects. The workflow resumes. On completion the card transitions to `COMPLETED` with a `TaskOutcome` showing the final status, a summary of what was accomplished, and the total action count.
8. At any point the operator can click **Halt** on any running card. The entity transitions immediately to `HALTED` and the workflow terminates.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, list, get, confirm, halt, SSE; serves `/api/metadata/*`. | — | `TaskEntity`, `TaskView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `TaskEntity` | `EventSourcedEntity` | Per-task lifecycle: submitted → running → (awaiting-confirmation) → completed / halted / failed. Source of truth. | `TaskEndpoint`, `TaskExecutionWorkflow` | `TaskView` |
| `TaskExecutionWorkflow` | `Workflow` | One workflow per task. Steps: `startStep` → `executeActionStep` (loops) → `awaitConfirmationStep` (when triggered) → `completeStep`. | started by `TaskEndpoint` on submit | `ComputerUseAgent`, `TaskEntity` |
| `ComputerUseAgent` | `AutonomousAgent` | The one decision-making LLM. Each iteration: receives the task description and the current screenshot as a task attachment; returns the next `ToolCall` or a `TaskOutcome`. | invoked by `TaskExecutionWorkflow` | returns `ToolCall` or `TaskOutcome` |
| `ActionGuardrail` | (before-tool-call hook on `ComputerUseAgent`) | Evaluates every proposed `ToolCall` before execution. Blocks destructive or out-of-scope actions and returns a structured rejection to the agent loop. | agent loop | agent loop |
| `ConfirmationGateway` | `Consumer` | Subscribes to `ConfirmationPending` events; records metadata for the UI; does NOT confirm autonomously. | `TaskEntity` events | `TaskView` |
| `TaskView` | `View` | Read model: one row per task for the UI. | `TaskEntity` events | `TaskEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum ActionType {
    SCREENSHOT, CLICK, TYPE, SCROLL, KEY_PRESS,
    NAVIGATE, FILE_READ, FILE_WRITE, FILE_DELETE,
    FORM_SUBMIT, ACCOUNT_MODIFY, API_CALL
}

enum ActionStatus { ALLOWED, BLOCKED, AWAITING_CONFIRMATION, CONFIRMED, REJECTED }

record ToolCall(
    String callId,
    ActionType actionType,
    String targetSelector,    // CSS selector, file path, URL, or key combo
    String payload,           // text to type, URL to navigate, or JSON body for API_CALL
    String rationale          // agent's one-line reason for the action
) {}

record ActionRecord(
    String callId,
    ActionType actionType,
    String targetSelector,
    String payload,
    ActionStatus status,
    String blockedReason,     // non-empty only when status == BLOCKED
    Instant executedAt
) {}

record ScreenshotAttachment(
    String screenshotId,
    byte[] imageBytes,
    String mimeType,          // "image/png"
    Instant capturedAt
) {}

record TaskOutcome(
    OutcomeStatus status,
    String summary,
    int totalActionsExecuted,
    int actionsBlocked,
    int actionsConfirmed,
    Instant completedAt
) {}
enum OutcomeStatus { SUCCESS, PARTIAL, FAILED, HALTED }

record ConfirmationRequest(
    String confirmationId,
    ToolCall pendingAction,
    String riskRationale,
    Instant requestedAt
) {}

record ConfirmationResponse(
    String confirmationId,
    boolean approved,
    String respondedBy,
    Instant respondedAt
) {}

record TaskRequest(
    String taskId,
    String description,
    String templateId,        // null when free-form
    Instant submittedAt,
    String submittedBy
) {}

record Task(
    String taskId,
    Optional<TaskRequest> request,
    List<ActionRecord> actionHistory,
    Optional<ConfirmationRequest> pendingConfirmation,
    Optional<TaskOutcome> outcome,
    TaskStatus status,
    boolean haltRequested,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus {
    SUBMITTED, RUNNING, AWAITING_CONFIRMATION,
    COMPLETED, HALTED, FAILED
}
```

Events on `TaskEntity`: `TaskSubmitted`, `TaskStarted`, `ActionProposed`, `ActionExecuted`, `ConfirmationPending`, `ConfirmationReceived`, `TaskCompleted`, `TaskHalted`, `TaskFailed`.

Every nullable lifecycle field on the `Task` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/tasks` — body `{ description, templateId?, submittedBy }` plus optional multipart screenshot → `{ taskId }`.
- `GET /api/tasks` — list all tasks, newest-first.
- `GET /api/tasks/{id}` — one task with full action history.
- `POST /api/tasks/{id}/halt` — operator halt; no body → `204`.
- `POST /api/tasks/{id}/confirm` — body `{ confirmationId, approved, respondedBy }` → `204`.
- `GET /api/tasks/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: ComputerUseAgent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted tasks (status pill + outcome badge + age) and a right pane with the selected task's detail — task description, scrollable action log with per-action status chips, the current screenshot (refreshed on each `ActionExecuted` event), a pending-confirmation banner when applicable, and a halt button.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call`, applied inside `ActionGuardrail` on `ComputerUseAgent`): runs before every action the agent proposes. Evaluates `ToolCall.actionType` against a blocklist (`FILE_DELETE`, `ACCOUNT_MODIFY` without confirmation), checks `targetSelector` against a domain allow-list for `NAVIGATE` and `API_CALL` actions, and checks `payload` against a content policy for `TYPE` and `FORM_SUBMIT` actions. On any failure returns a structured `BLOCKED` rejection to the agent loop; the agent may propose an alternative within its iteration budget. Allowed actions flow through to the environment simulator.
- **H1 — operator halt** (`operator-regulator-stop`, applied as `TaskHalted` event + halt-flag check in `TaskExecutionWorkflow`): an operator POSTs to `/api/tasks/{id}/halt`. `TaskEndpoint` calls `TaskEntity.halt()`, which emits `TaskHalted` and sets `haltRequested = true`. The workflow's `executeActionStep` checks the halt flag at the start of every iteration; on `haltRequested == true` it transitions immediately to the terminal `haltStep` without executing any further actions and without querying the agent.
- **C1 — human-in-the-loop confirmation** (`hitl`, applied as `ConfirmationPending` / `ConfirmationReceived` event pair + `awaitConfirmationStep` in `TaskExecutionWorkflow`): high-impact action types (`FORM_SUBMIT`, `FILE_WRITE`, `FILE_DELETE`, `ACCOUNT_MODIFY`, `API_CALL`) trigger a confirmation gate before execution. The workflow writes `ConfirmationPending` and enters `awaitConfirmationStep`, polling `TaskEntity.getTask` every 2 s up to a 5-minute timeout. The user responds via the UI or the `/confirm` endpoint; `TaskEndpoint` calls `TaskEntity.recordConfirmation(response)`, which emits `ConfirmationReceived`. On approval the workflow executes the action; on rejection it writes `ActionExecuted` with `status = REJECTED` and returns a `USER_REJECTED` signal to the agent for recovery.

## 9. Agent prompts

- `ComputerUseAgent` → `prompts/computer-use-agent.md`. The single decision-making LLM. System prompt instructs it to observe the current screenshot, select the next action from the allowed action type set, and return a `ToolCall`; or to return a `TaskOutcome` when the task is complete or unrecoverable.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the fill-web-form seed task; within 60 s the task completes with `OutcomeStatus.SUCCESS`, every action in the log has `status == ALLOWED`, and the action count matches the number of form fields.
2. **J2** — The agent proposes a `FILE_DELETE` action; the before-tool-call guardrail blocks it with reason `BLOCKED_DESTRUCTIVE_ACTION`; the agent proposes an alternative; the task completes without the deletion executing.
3. **J3** — The agent proposes a `FORM_SUBMIT` action; the entity transitions to `AWAITING_CONFIRMATION`; the user approves; the submission executes and the task continues.
4. **J4** — While a task is in `RUNNING` state, the operator posts to `/halt`; the entity transitions to `HALTED`; the action history is intact; no further actions execute.
5. **J5** — The user rejects a `FILE_WRITE` confirmation; the agent receives `USER_REJECTED`, adjusts its plan, and completes the task via an alternative path (`FILE_READ` only).

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named computer-use-agent demonstrating the single-agent × ops-automation cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-ops-automation-computer-use-agent. Java package io.akka.samples.computeruseagent.
Akka 3.6.0. HTTP port 9391.

Components to wire (exactly):

- 1 AutonomousAgent ComputerUseAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/computer-use-agent.md>) and
  .capability(TaskAcceptance.of(EXECUTE_TASK).maxIterationsPerTask(20)). Each task
  invocation receives the task description in the task's instruction text and the current
  screenshot as a task ATTACHMENT named "screen.png" (TaskDef.attachment("screen.png",
  imageBytes) — NOT as inline prompt text). The agent returns either a ToolCall (next action)
  or a TaskOutcome (terminal). The agent is configured with a before-tool-call guardrail (see
  G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block. On
  guardrail rejection the agent loop receives the structured BLOCKED reason and may propose an
  alternative within its maxIterationsPerTask budget.

- 1 Workflow TaskExecutionWorkflow per taskId with these steps:
  * startStep — calls TaskEntity.markRunning(), then advances to executeActionStep.
    WorkflowSettings.stepTimeout 5s.
  * executeActionStep — checks TaskEntity.getTask().haltRequested(); if true, transitions
    to haltStep. Otherwise calls componentClient.forAutonomousAgent(
    ComputerUseAgent.class, "agent-" + taskId).runSingleTask(
      TaskDef.instructions(task.request.description)
        .attachment("screen.png", currentScreenshot.imageBytes)
    ). On the task result: if ToolCall, runs ActionGuardrail.evaluate(toolCall); if ALLOWED,
    calls EnvironmentSimulator.execute(toolCall) and records ActionExecuted(status=ALLOWED);
    if BLOCKED, records ActionExecuted(status=BLOCKED, blockedReason=...) and re-runs
    executeActionStep with the rejection returned to the agent; if toolCall is high-impact
    (FORM_SUBMIT, FILE_WRITE, FILE_DELETE, ACCOUNT_MODIFY, API_CALL), writes
    ConfirmationPending and transitions to awaitConfirmationStep. If TaskOutcome, transitions
    to completeStep. WorkflowSettings.stepTimeout 30s with defaultStepRecovery
    maxRetries(2).failoverTo(TaskExecutionWorkflow::errorStep).
  * awaitConfirmationStep — polls TaskEntity.getTask every 2s; on
    task.pendingConfirmation == null (cleared by ConfirmationReceived), reads the last
    ConfirmationResponse from the entity. If approved, executes the action and returns to
    executeActionStep. If rejected, records ActionExecuted(status=REJECTED) and returns to
    executeActionStep with USER_REJECTED signal. WorkflowSettings.stepTimeout 300s (5 min
    for user to respond).
  * completeStep — calls TaskEntity.complete(outcome). WorkflowSettings.stepTimeout 5s.
  * haltStep — calls TaskEntity.halt(). WorkflowSettings.stepTimeout 5s.
  * errorStep — calls TaskEntity.fail(reason). WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity TaskEntity (one per taskId). State Task{taskId: String,
  request: Optional<TaskRequest>, actionHistory: List<ActionRecord>,
  pendingConfirmation: Optional<ConfirmationRequest>,
  outcome: Optional<TaskOutcome>, status: TaskStatus, haltRequested: boolean,
  createdAt: Instant, finishedAt: Optional<Instant>}. TaskStatus enum: SUBMITTED, RUNNING,
  AWAITING_CONFIRMATION, COMPLETED, HALTED, FAILED. Events: TaskSubmitted{request},
  TaskStarted{}, ActionProposed{toolCall}, ActionExecuted{actionRecord},
  ConfirmationPending{confirmationRequest}, ConfirmationReceived{confirmationResponse},
  TaskCompleted{outcome}, TaskHalted{reason}, TaskFailed{reason}.
  Commands: submit, markRunning, recordAction, requestConfirmation, recordConfirmation,
  complete, halt, fail, getTask. emptyState() returns Task.initial("") with all Optional
  fields as Optional.empty(), actionHistory as Collections.emptyList(), haltRequested false,
  and no commandContext() reference (Lesson 3).

- 1 Consumer ConfirmationGateway subscribed to TaskEntity events; on ConfirmationPending
  enriches the task view with the pending action's riskRationale so the UI can render the
  confirmation banner. Does NOT autonomously approve or reject — it is a read-side
  projector only. No write-back to TaskEntity.

- 1 View TaskView with row type TaskRow (mirrors Task minus the raw screenshot bytes).
  Table updater consumes TaskEntity events. ONE query getAllTasks: SELECT * AS tasks FROM
  task_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * TaskEndpoint at /api with:
    POST /tasks (body {description, templateId?, submittedBy}; mints taskId; calls
      TaskEntity.submit; starts TaskExecutionWorkflow; returns {taskId}),
    GET /tasks (list from getAllTasks, sorted newest-first),
    GET /tasks/{id} (one row with full actionHistory),
    POST /tasks/{id}/halt (calls TaskEntity.halt(); returns 204),
    POST /tasks/{id}/confirm (body {confirmationId, approved, respondedBy};
      calls TaskEntity.recordConfirmation; returns 204),
    GET /tasks/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Supporting classes:

- TasksTasks.java declaring one Task<R> constant: EXECUTE_TASK = Task.name("Execute task")
  .description("Observe the current screen and return the next ToolCall, or return a
  TaskOutcome if the task is complete or unrecoverable")
  .resultConformsTo(ToolCall.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- ActionGuardrail.java implementing the before-tool-call hook. Evaluates every proposed
  ToolCall against three checks: (1) actionType blocklist (FILE_DELETE and ACCOUNT_MODIFY
  without a prior ConfirmationReceived), (2) domain allow-list for NAVIGATE/API_CALL
  targetSelector, (3) content policy for TYPE/FORM_SUBMIT payload. Returns
  Guardrail.reject(<structured-BLOCKED-reason>) on any failure or Guardrail.pass() to allow.

- EnvironmentSimulator.java — in-process stub that records the action and returns a new
  ScreenshotAttachment representing the state after the action. Seeded with 3 scenarios
  matching the task templates. Pure deterministic logic; no external process.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9391 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-tasks.jsonl with 3 seeded task templates:
  fill-web-form (fill a 4-field contact form), navigate-and-extract (open a product page
  and extract the title and price), bulk-rename-files (rename 3 files in a directory with
  a date prefix). Each template includes an expectedActionSequence list for the mock LLM.

- src/main/resources/sample-events/seed-screenshots.jsonl with 3 synthetic base64-encoded
  PNG screenshots (minimal 64x64 placeholders) paired to each template.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, C1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root pre-filled for the ops-automation domain.

- prompts/computer-use-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: ComputerUseAgent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of task cards with status pill, outcome badge, age; right = selected-task detail
  with task description, action log table, screenshot panel, pending-confirmation banner,
  halt button). Browser title exactly: <title>Akka Sample: ComputerUseAgent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns shape-correct
        ToolCall or TaskOutcome objects from src/main/resources/mock-responses/
        execute-task.json. The mock selects entries deterministically by taskId modulo the
        entry count. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface. For EXECUTE_TASK,
  reads src/main/resources/mock-responses/execute-task.json and returns entries in sequence,
  cycling when exhausted. Entries include:
    - 8 ToolCall entries covering CLICK, TYPE, NAVIGATE, SCREENSHOT, FORM_SUBMIT, FILE_READ,
      FILE_WRITE actions with realistic targetSelector and payload values matching the three
      seed templates. At least 2 entries are high-impact (FORM_SUBMIT, FILE_WRITE) to trigger
      the confirmation gate. At least 1 entry is FILE_DELETE to trigger the guardrail.
    - 3 TaskOutcome entries covering SUCCESS, PARTIAL, FAILED outcomes with realistic summary
      texts. The mock cycles through action entries until it returns a TaskOutcome to simulate
      a complete multi-step execution.
  MockModelProvider.seedFor(taskId) selects the starting index deterministically.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ComputerUseAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion TasksTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (startStep 5s,
  executeActionStep 30s, awaitConfirmationStep 300s, completeStep 5s, haltStep 5s,
  errorStep 5s).
- Lesson 6: every nullable lifecycle field on the Task row record is Optional<T>.
- Lesson 7: TasksTasks.java with EXECUTE_TASK is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9391 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in any user-facing prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: exactly ONE AutonomousAgent (ComputerUseAgent). The
  confirmation gateway (ConfirmationGateway) is a read-side Consumer — it does NOT call a
  model.
- The screenshot is passed as a task ATTACHMENT, never inlined into the agent's instruction
  text. The generated executeActionStep uses TaskDef.attachment("screen.png", ...).
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check after the agent returns.
- The halt is unconditional — the workflow checks haltRequested at the START of
  executeActionStep before any agent call. The agent is never queried when halt is set.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
