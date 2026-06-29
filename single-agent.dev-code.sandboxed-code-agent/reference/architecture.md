# Architecture — sandboxed-code-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call with a tool attached. `ExecutionEndpoint` accepts a task submission, writes a `CodeSubmitted` event onto `ExecutionEntity`, and starts an `ExecutionWorkflow` instance. The workflow's `screenStep` calls `CodeExecutionAgent` — the single AutonomousAgent — with the task description as its instruction text. When the agent calls the `execute_code` tool, `CodeScreeningGuardrail` intercepts the call as a `before-tool-call` hook. The guardrail either passes the code through to the sandbox back-end or rejects it with a named pattern, forcing the agent to revise. Once the agent returns an approved `ExecutionResult`, the workflow emits `CodeApproved` onto the entity.

`SandboxRouter` subscribes to `CodeApproved` events. It reads the approved code and sandbox configuration, dispatches to the back-end configured by `SANDBOX_BACKEND` (`DockerSandboxDispatcher`, `E2BSandboxDispatcher`, `ModalSandboxDispatcher`, or `PlaywrightSandboxDispatcher`), collects the `SandboxOutput`, and calls `ExecutionEntity.recordOutcome`.

In parallel, `SafetyHaltMonitor` subscribes to `ExecutionStarted` events and polls the sandbox back-end's resource channel every 2 seconds. On a budget breach it kills the process and calls `ExecutionEntity.halt(haltReason)`.

`ExecutionView` projects every entity event into a read-model row. `ExecutionEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. `SafetyHaltMonitor` and `SandboxRouter` are Consumers — they make no LLM calls. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system waits on something:

1. The `before-tool-call` interception inside the agent loop — sub-millisecond for pattern matching; only adds latency if the agent retries after a block.
2. The `runStep` timeout — sized to `wallClockBudgetSeconds + 10` seconds. The sandbox's actual runtime is bounded by the budget; `runStep`'s timeout is a safety margin over that.

The `SafetyHaltMonitor` polling loop runs concurrently with `runStep`. In the happy path it polls a few times and then cancels when `ExecutionCompleted` lands. In the halt path it fires before `runStep` times out, killing the sandbox and landing `ExecutionHalted` on the entity first.

## State machine

Seven states. The paths of interest:

- Happy path: `SUBMITTED → SCREENING → APPROVED → RUNNING → COMPLETED`.
- Guardrail retry: the entity stays in `SCREENING` each time `CodeBlocked` lands (the agent loop is still running). Once the agent succeeds, `CodeApproved` advances to `APPROVED`.
- Budget breach: `RUNNING → HALTED` via `ExecutionHalted` from `SafetyHaltMonitor`.
- Guardrail exhaustion: if all 4 agent iterations are blocked, `screenStep` fails over to `error` → `FAILED`.
- Sandbox error: `RUNNING → FAILED` if the dispatcher throws and `SandboxRouter` calls `fail(reason)`.

There is no `APPROVED_BY_HUMAN` state. The guardrail is machine-enforced; the system is fully autonomous within its pattern-match rules. An operator who needs a human-approval gate would add a `waitForApprovalStep` to the workflow between `screenStep` and `runStep`.

## Entity model

`ExecutionEntity` is the source of truth. It emits eight event types. `ExecutionView` projects every event into a row used by the UI. `SandboxRouter` subscribes to `CodeApproved` to drive the actual sandbox dispatch. `SafetyHaltMonitor` subscribes to `ExecutionStarted` to start the resource polling loop. `ExecutionWorkflow` both reads (`getExecution`) and writes (`markRunning`, `recordOutcome`, `fail`) on the entity. `CodeExecutionAgent`'s relationship to `ExecutionResult` is "returns" — the agent's task result is the typed record.

## Defence-in-depth governance flow

For any execution that enters the `RUNNING` state, the code passed through:

1. **before-tool-call guardrail** — five pattern-match passes block forbidden code before the sandbox API is called.
2. **Sandbox isolation** — Docker runs with `--network none`, `--memory 256m`, `--cpus 0.5`, and a tmpfs mount. E2B and Modal enforce equivalent cloud-level isolation.
3. **Resource monitor** — wall-clock and CPU budgets are enforced independently of the agent loop by `SafetyHaltMonitor`; the agent cannot extend its own budget.

Each layer is independent. Removing any one opens an explicit gap the others do not cover: the guardrail catches forbidden patterns before they run; the sandbox limits blast radius if a pattern slips through; the monitor kills processes that comply with the guardrail but still run forever.
