# Architecture — computer-use-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call per action step. `TaskEndpoint` accepts a task submission, writes a `TaskSubmitted` event onto `TaskEntity`, and starts `TaskExecutionWorkflow`. The workflow's `executeActionStep` calls `ComputerUseAgent` — the single AutonomousAgent — with the task description as `TaskDef.instructions(...)` and the current screenshot as a `TaskDef.attachment("screen.png", imageBytes)`. Before any action reaches the environment, `ActionGuardrail` (the before-tool-call hook) evaluates it against the action policy. Allowed tool calls proceed to `EnvironmentSimulator.execute(toolCall)`, which returns an updated `ScreenshotAttachment`; the cycle repeats. When the agent returns a `TaskOutcome`, the workflow transitions to `completeStep`.

Two interruption paths cross the execution loop. The operator halt path — `POST /api/tasks/{id}/halt` → `TaskEntity.halt()` → `haltRequested = true` — is checked at the top of every `executeActionStep` iteration before any agent call. The confirmation gate path — high-impact `ToolCall` detected → `ConfirmationPending` event → `awaitConfirmationStep` → user responds → `ConfirmationReceived` — suspends the loop until explicit approval or rejection arrives.

`ConfirmationGateway` is a read-side Consumer that subscribes to `ConfirmationPending` events and enriches `TaskView` with the pending action's risk rationale. It writes nothing back to `TaskEntity`.

`TaskView` projects every entity event into a read-model row. `TaskEndpoint` serves the read model over REST and SSE.

The graph deliberately has no second agent. Governance is applied through three mechanisms — a pre-execution guardrail, an unconditional halt, and a human confirmation gate — none of which require a second LLM call. That is what makes this a faithful **single-agent** blueprint.

## Interaction sequence

The sequence traces the fill-web-form happy path (J1). Three notable moments:

1. `executeActionStep` calls the agent with the current screenshot attached. The agent observes the screen and proposes one `ToolCall`.
2. `ActionGuardrail` evaluates the `ToolCall` before it reaches the environment. On the happy path every action is `ALLOWED`.
3. `EnvironmentSimulator` executes the action and returns a new screenshot. The loop continues until the agent returns a `TaskOutcome`.

The FORM_SUBMIT confirmation path (J3) inserts an `awaitConfirmationStep` between the proposed action and its execution — the loop pauses until the user responds.

## State machine

Six states. The interesting transitions:

- `RUNNING → AWAITING_CONFIRMATION → RUNNING` is the round-trip for a confirmed high-impact action. The entity returns to `RUNNING` regardless of the user's decision; the outcome of their decision (approved or rejected) is encoded in the `ActionRecord` written to `actionHistory`.
- `RUNNING → HALTED` and `AWAITING_CONFIRMATION → HALTED` are unconditional; both require only the `TaskHalted` event triggered by the halt endpoint.
- There is no `COMPLETED → RUNNING` transition. A completed task's record is immutable. Retrying a task requires submitting a new one.
- `FAILED` captures two different failure sources: agent error (guardrail exhaustion, LLM timeout) and confirmation timeout (user did not respond within 300 s). Both land in `FAILED`; the `reason` field on `TaskFailed` distinguishes them.

## Entity model

`TaskEntity` is the source of truth. It emits nine event types. `TaskView` projects every event. `ConfirmationGateway` subscribes to `ConfirmationPending` only. `TaskExecutionWorkflow` both reads (`getTask`, checking `haltRequested` and `pendingConfirmation`) and writes (`markRunning`, `recordAction`, `requestConfirmation`, `recordConfirmation`, `complete`, `halt`, `fail`). `ComputerUseAgent` returns either a `ToolCall` or a `TaskOutcome`; neither is an event on its own — the workflow translates them into entity commands.

## Defence-in-depth governance flow

For any action that executes in the environment, it has passed through:

1. **before-tool-call guardrail** — the action type, target, and payload are evaluated against the policy ruleset before the action touches the environment.
2. **Halt check** — `executeActionStep` reads `haltRequested` before every agent call and every action execution; a halt issued at any point stops the next action from firing.
3. **Confirmation gate** — high-impact action types pause for explicit user approval; the action does not execute on timeout or rejection.

Each mechanism is independent. The guardrail does not know about pending confirmations. The halt does not wait for the guardrail. The confirmation gate does not require the guardrail to have fired first.
