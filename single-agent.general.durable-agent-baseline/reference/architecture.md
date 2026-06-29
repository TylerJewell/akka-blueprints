# Architecture — durable-agent-baseline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one durable execution loop. `WorkOrderEndpoint` accepts a submission, writes a `WorkOrderInitiated` event onto `WorkOrderEntity`, and starts a `WorkOrderWorkflow` instance. The workflow's `initStep` marks the entity running and hands off to `agentStep`, which calls `WorkOrderAgent` — the single AutonomousAgent — with the work order's steps as `TaskDef.instructions(...)`. The agent processes each step and returns a `WorkOrderResult`. The workflow writes `WorkOrderCompleted` and runs `PerformanceEvaluator` in `evalStep`.

In parallel, `RuntimeMonitor` subscribes to the entity's event stream. It opens a timed window on each `StepStarted` event and closes it on `StepCompleted` or `StepFailed`. An agent that takes longer than 90 s on a single step causes the monitor to call `WorkOrderEntity.recordAlert` with `kind = STALL`. This alert lands in the entity log and appears in the UI without interrupting the agent.

`WorkOrderView` projects every entity event into a read-model row. `WorkOrderEndpoint` serves that view over REST and SSE.

The graph deliberately has no second agent. `PerformanceEvaluator` is a deterministic rule-based scorer — not an LLM. That is what keeps this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two key moments:

1. The `agentStep` is where LLM latency accumulates. The step's 120 s timeout accommodates a 5-step work order with up to 5 agent iterations, each taking up to ~20 s.
2. `RuntimeMonitor`'s stall detection runs on a 30 s scan cycle. For long steps the first stall alert fires at the first scan crossing the 90 s threshold — not exactly at 90 s. This is intentional: the monitor is a human-on-loop signal, not a hard circuit breaker.

## State machine

Eight states. Key paths:

- The happy path is `INITIATED → RUNNING → STEP_COMPLETED (×N) → COMPLETED → EVALUATED`.
- `STALLED` is a lateral state: the entity enters it when an `AlertRecorded (STALL)` event lands during `RUNNING`, and returns to `RUNNING` if the agent eventually completes the step.
- Two terminal failure states: `FAILED` from `RUNNING` (agent error or step exhaustion) and `FAILED` from `STALLED` (stall + subsequent failure).
- There is no `APPROVED` or `CLOSED` state. The work order result is the agent's output; downstream action happens outside the system.

## Entity model

`WorkOrderEntity` is the source of truth, emitting nine event types across the lifecycle. `WorkOrderView` projects every event into a row for the UI. `RuntimeMonitor` subscribes to entity events and writes alert events back. `WorkOrderWorkflow` both reads (`getWorkOrder`) and writes (`markRunning`, `recordResult`, `recordEvaluation`, `fail`) on the entity. The relationship between `WorkOrderAgent` and `WorkOrderResult` is "returns" — the agent's task result is the result record.

## Durability guarantee

For any work order that reaches `EVALUATED`, the execution passed through two independent durability mechanisms:

1. **Workflow step journaling** — the Akka Workflow checkpoints its position after each step boundary. A JVM restart leaves the workflow at its last committed step; `initStep` and `evalStep` re-run cleanly (they are idempotent), but `agentStep` resumes rather than re-executing from scratch.
2. **Entity event log** — every step transition (started, completed, failed) is an immutable event. After restart, `RuntimeMonitor` re-reads the entity's current state to reconstruct its stall-window tracking without double-firing alerts.

Each durability mechanism is independent. A failure in the monitor does not affect workflow progress, and a workflow failure does not corrupt the entity event log.
