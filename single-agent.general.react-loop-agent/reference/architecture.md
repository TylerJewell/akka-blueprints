# Architecture — react-loop-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM running a ReAct loop. `RunEndpoint` accepts a query submission, writes a `RunSubmitted` event onto `RunEntity`, and starts a `ReActWorkflow` instance. The workflow's `initStep` marks the run as RUNNING; then `loopStep` begins calling `ReActAgent` — the single AutonomousAgent — iteratively.

On each iteration the agent emits one Step. If the step is a THOUGHT or OBSERVATION, the workflow records it directly on the entity. If the step is an ACTION, the workflow calls `ToolCallGuardrail` first. A blocked action is recorded as `ActionBlocked` on the entity and a synthetic OBSERVATION is returned to the agent; the agent sees the block reason and decides how to proceed. An allowed action is recorded as a `StepRecorded` event with `kind=ACTION`, which the `ToolDispatcher` Consumer picks up, executes the named tool stub, and writes back an OBSERVATION step. When the agent emits an ANSWER step, the workflow transitions to `finalizeStep`, records `RunCompleted`, calls `ChainEvaluator` to score the trace, and writes `EvaluationScored`.

`RunView` projects every entity event into a read-model row that `RunEndpoint` serves to the UI over REST and SSE.

The graph has no second agent. `ToolCallGuardrail` and `ChainEvaluator` are deterministic Java classes — not LLM calls. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two important pauses in the flow:

1. **Per-iteration agent call**: the agent does not return all steps at once. Each call to `loopStep` retrieves one Step. The workflow loops back into the agent on the next iteration, forwarding the accumulated step context.
2. **ToolDispatcher lag**: between an `ActionRequested` event landing and the `ObservationRecorded` call completing, the Consumer executes the tool stub. In-process stubs complete in milliseconds; a real external integration would see latency here and should size `loopStep`'s timeout accordingly.

## State machine

Five states. Key paths:

- The happy path is `PENDING → RUNNING → COMPLETED`, then `EvaluationScored` appended (no status change from COMPLETED).
- Budget exhaustion: `RUNNING → EXHAUSTED` when `stepIndex >= 10` with no ANSWER. The partial trace is preserved; eval still runs.
- Failure: `RUNNING → FAILED` on an unrecoverable workflow error. The partial trace is preserved; eval does not run on a FAILED entity.
- There is no APPROVED or REJECTED state. The run result is the agent's final answer; what the user does with it is outside the system.

## Entity model

`RunEntity` is the source of truth. It emits eight event types. `RunView` projects every event into a row used by the UI. `ToolDispatcher` subscribes to entity events to execute tool stubs and write back observations. `ReActWorkflow` both reads (`getRun`) and writes (`markRunning`, `recordStep`, `blockAction`, `complete`, `exhaust`, `recordEvaluation`, `fail`) on the entity. The relationship between `ReActAgent` and `ReActResult` is "returns" — the agent's final task result is the result record.

## Governance flow

For any run that reaches COMPLETED or EXHAUSTED:

1. **before-tool-call guardrail** — `ToolCallGuardrail` prevents unauthorized, schema-invalid, or denied tool calls from reaching the stubs. Blocked calls are fully auditable as `ActionBlocked` events.
2. **ReActAgent** — one model running a bounded iteration loop. The iteration cap (10) is enforced by the workflow; the agent cannot extend its own budget.
3. **On-decision chain evaluator** — `ChainEvaluator` scores the finished trace for circular loops, hallucinated tool claims, and thought-only chains. The score gives operators a fast signal about which completed runs warrant inspection.

Each step is independent: the tool guardrail does not know about the chain evaluator; the chain evaluator does not modify the run's terminal state. Removing one opens a gap the other does not silently cover.
