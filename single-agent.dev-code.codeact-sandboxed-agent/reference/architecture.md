# Architecture — codeact-sandboxed-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `TaskEndpoint` accepts a submission, writes a `TaskSubmitted` event onto `TaskEntity`, and starts an `ExecutionWorkflow`. The workflow's `executeStep` calls `CodeActAgent` — the single AutonomousAgent — with the task description and acceptance criterion as `TaskDef.instructions(...)` and the context data as a `TaskDef.attachment(...)`. Every `execute_code` tool call the agent emits is intercepted by `SandboxGuardrail` (the `before-tool-call` hook); code that passes runs through the in-process sandbox. After each sandbox run, `ExecutionWorkflow` calls `SafetyHaltMonitor` synchronously; on a halt signal it calls `TaskEntity.triggerHalt(reason)` and stops. Clean output triggers a `CodeExecuted` event. The `SecretSanitizer` Consumer subscribes to `CodeExecuted`, scrubs credential patterns from the raw output, and writes the sanitized form back via `attachSanitizedOutput`. The workflow's `inspectStep` then calls `AcceptanceChecker`; if the criterion is met, it calls `TaskEntity.markSolved`. Otherwise the workflow loops back to `executeStep` until the iteration budget is exhausted.

The graph has no second agent. `AcceptanceChecker`, `SafetyHaltMonitor`, and `SandboxGuardrail` are deterministic rule-based classes. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Key moments where the system pauses:

1. The `before-tool-call` guardrail intercepts every `execute_code` call synchronously — if rejected, the agent loop rewrites and retries within the same task.
2. `SafetyHaltMonitor` runs synchronously inside `executeStep` after the sandbox returns — a halt fires immediately, before the `CodeExecuted` event is written.
3. The `SecretSanitizer` Consumer processes `CodeExecuted` asynchronously — sub-second in normal operation; the workflow's `inspectStep` reads from the entity after the sanitized form has landed.

The agent call is bounded by `executeStep`'s 60 s timeout to accommodate LLM latency plus sandbox execution time. The `inspectStep` is bounded at 15 s. Terminal steps are bounded at 5 s each.

## State machine

Five states. The notable paths:

- The happy path is `SUBMITTED → EXECUTING → SOLVED`. The `EXECUTING` state re-enters itself once per code-execution iteration; each re-entry corresponds to one `CodeExecuted` event.
- `HALTED` is a distinct terminal state reached only via `HaltTriggered`. It is not the same as `FAILED` — a halted task has a specific security reason; a failed task ran out of iterations or encountered an agent error.
- `FAILED` covers two scenarios: budget exhaustion (the agent iterated 5 times without satisfying the acceptance criterion) and agent errors (the workflow's error step fires after two step retries).
- All three terminal states (`SOLVED`, `HALTED`, `FAILED`) preserve the full iteration history on the entity. The UI shows the complete per-iteration log regardless of how the task ended.

## Entity model

`TaskEntity` is the source of truth. It emits six event types. `TaskView` projects every event into a row used by the UI. `SecretSanitizer` subscribes to `CodeExecuted` events to produce the sanitized output. `ExecutionWorkflow` both reads (`getTask`) and writes (`recordExecution`, `attachSanitizedOutput`, `triggerHalt`, `markSolved`, `markFailed`) on the entity. The relationship between `CodeActAgent` and `TaskResolution` is "returns" — the agent's task result is the resolution record, which the workflow then applies to the entity.

## Defence-in-depth governance flow

For any output that reaches the `SOLVED` state, the execution passed through:

1. **Before-tool-call guardrail** — static code analysis runs before the sandbox receives the code; forbidden patterns are caught before execution begins.
2. **Sandbox execution** — the in-process restricted executor runs the code with no network, no filesystem writes, and a configurable timeout.
3. **Safety halt monitor** — every output is scanned for destructive action signatures and secret-like tokens before the `CodeExecuted` event is written.
4. **Secret sanitizer** — every output that passes the halt check is scrubbed for credential patterns before it lands in the view or the UI.
5. **Acceptance checker** — the task is only marked `SOLVED` when the sanitized output satisfies the declared acceptance criterion.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
