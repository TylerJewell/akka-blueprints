# Architecture — code-interpreter-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `JobEndpoint` accepts a job submission, creates a `JobEntity`, and starts an `InterpretationWorkflow` instance. The workflow's `guardStep` calls `CodeInterpreterAgent` — the single AutonomousAgent — with the user prompt as `TaskDef.instructions(...)` and the data payload as a `TaskDef.attachment(...)`. The agent emits a `code-execution` tool call containing the generated Python source. The `CodeGuardrail`, wired to the `before-tool-call` hook, inspects the source before any code runs. Approved code flows to `executeStep`, where `ExecutionSandbox` runs it under wall-clock and memory budgets. The sandbox result becomes an `ExecutionResult`, which the workflow writes back to `JobEntity` via `recordResult`. For breach conditions, the sandbox path calls `JobEntity.halt(breachType)` instead. `JobView` projects every entity event into a read-model row; `JobEndpoint` serves the read model over REST and SSE.

The graph has no second agent. `ExecutionSandbox` is a plain Java class — deterministic, in-process, no LLM call — which is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct checkpoints where execution can be interrupted:

1. The `before-tool-call` guardrail evaluation inside `guardStep` — sub-millisecond in normal operation, but may repeat up to 3 times if the agent's first code attempts are rejected.
2. The sandbox watchdog timer inside `executeStep` — fires after 10 seconds if the code has not completed. Normal computations complete in under 100 ms.

The agent call itself is bounded by `guardStep`'s 60 s timeout, which covers both the LLM generation time and up to 3 guardrail-triggered retries.

## State machine

Six states. The notable paths:

- The happy path is `SUBMITTED → CODE_APPROVED → EXECUTING → RESULT_RECORDED`.
- `HALTED` is a terminal state that can only be reached from `EXECUTING` when the sandbox watchdog fires. A halted job's `JobHalted` event records the breach type (`TIMED_OUT` or `MEMORY_EXCEEDED`) and elapsed wall-clock time.
- `FAILED` is reachable from both `SUBMITTED` (workflow start failure) and `CODE_APPROVED` (if the agent exhausts its 3-iteration retry budget without producing approved code). A failed job preserves whatever state was recorded before the failure — `generatedCode` may be populated even on `FAILED` if at least one code attempt was made before the guardrail exhausted iterations.
- There is no `APPROVED_BY_HUMAN` state. Code execution in this blueprint is fully automated; human oversight is a deployer-level decision surfaced in the risk survey.

## Entity model

`JobEntity` is the source of truth. It emits six event types. `JobView` projects every event into a row for the UI. `InterpretationWorkflow` both reads (`getJob`) and writes (`approveCode`, `markExecuting`, `recordResult`, `halt`, `fail`) on the entity. The relationship between `CodeInterpreterAgent` and `ExecutionResult` is "returns" — the agent's task result is the execution-result record. The relationship between `ExecutionSandbox` and `SandboxResult` is also "returns" — the sandbox output is an intermediate value consumed by the workflow before the final `ExecutionResult` is written to the entity.

## Defence-in-depth governance flow

For any job that reaches `RESULT_RECORDED`, the generated code passed through:

1. **CodeGuardrail (before-tool-call)** — the model's generated source was inspected for forbidden imports, shell escapes, out-of-scope file writes, and network calls before any code ran.
2. **ExecutionSandbox** — the approved code ran under a hard wall-clock budget (10 s) and a memory heuristic (128 MB). Any overrun terminates the execution before it can exhaust host resources.

Each control addresses a different failure mode. The guardrail catches intent-level violations (code that tries to do something forbidden); the halt catches resource-level violations (code that accidentally or adversarially loops). Removing either opens a gap the other cannot cover.
