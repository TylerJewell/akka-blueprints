# Architecture — durable-workflow-recovery

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `RecoveryEndpoint` accepts an execution registration and writes an `ExecutionRegistered` event onto `ExecutionEntity`. As checkpoint events arrive (from the seeded replay or a real upstream source), `CheckpointConsumer` monitors the inter-checkpoint gap. When the gap exceeds the stall timeout, the consumer builds a `CheckpointSnapshot`, writes it back via `markStalled`, and starts a `RecoveryWorkflow` instance.

The workflow's `analyzeStep` calls `WorkflowRecoveryAgent` — the single AutonomousAgent — with the execution metadata as `TaskDef.instructions(...)` and the checkpoint snapshot serialised to JSON as a `TaskDef.attachment("checkpoint-snapshot.json", ...)`. The agent returns a `RecoveryDecision`. `ResponseValidator` checks the decision's structural integrity before it is committed to the entity. If the verdict is RESUME, a `ResumeCommand` is dispatched to restart the execution from its last durable checkpoint. Then `HealthEvaluator` runs in `healthEvalStep`, scoring the execution on latency drift, progress rate, and retry saturation. The score lands as `HealthScored`.

`ExecutionView` projects every entity event into a read-model row; `RecoveryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. `HealthEvaluator` is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct pauses:

1. The `CheckpointConsumer` stall-detection lag — the consumer fires on `CheckpointRecorded` events and checks the elapsed time since the previous checkpoint. The stall is detected on the first event after the configured timeout expires.
2. The `awaitStalledStep` polling loop inside the workflow — polls `ExecutionEntity` every 1 s up to its 30 s timeout, advancing as soon as `execution.snapshot().isPresent()` returns true.

The agent call is bounded by `analyzeStep`'s 60 s timeout. `healthEvalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Eight states. The interesting paths:

- The happy RESUME path is `REGISTERED → RUNNING → STALLED → ANALYZING → DECISION_RECORDED → HEALTH_SCORED`.
- An ABORT decision routes `ANALYZING → ABORTED` — the restart command is never dispatched.
- An ESCALATE decision routes through `DECISION_RECORDED → HEALTH_SCORED` and flags the card on the UI for human review.
- Two FAILED transitions: a system error during `RUNNING`, and a workflow error (validator rejection after all retries) during `STALLED`. A `FAILED` execution's partial checkpoint data is preserved on the entity — the UI shows the last known state.
- There is no `COMPLETED` state at the entity level. Completion is inferred by the upstream workflow system; this service is concerned only with recovery. Once `HEALTH_SCORED`, the execution record is terminal in this system.

## Entity model

`ExecutionEntity` is the source of truth. It emits eight event types. `ExecutionView` projects every event into a row used by the UI. `CheckpointConsumer` subscribes to entity events to detect stalls and drive periodic health ticks. `RecoveryWorkflow` both reads (`getExecution`) and writes (`startAnalysis`, `recordDecision`, `recordHealth`, `abort`, `fail`) on the entity. The relationship between `WorkflowRecoveryAgent` and `RecoveryDecision` is "returns" — the agent's task result is the decision record.

## Durability and recovery flow

For any decision that lands in the entity log, the checkpoint state passed through:

1. **Stall detection** — the `CheckpointConsumer` confirms no progress for longer than the configured timeout before declaring a stall.
2. **Snapshot capture** — all durable checkpoint records are bundled into a `CheckpointSnapshot` before the agent call; the agent never accesses live entity state directly.
3. **WorkflowRecoveryAgent** — one model call, one structured output, with the full checkpoint history as context.
4. **ResponseValidator** — structural checks before the decision is committed; a validator failure fails the workflow step rather than committing a partial decision.
5. **Periodic health eval** — every recorded decision is accompanied by a 1–5 health score so operators know which executions to inspect first.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
