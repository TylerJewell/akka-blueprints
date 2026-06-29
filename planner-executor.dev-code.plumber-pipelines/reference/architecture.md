# Architecture — plumber-data-engineering-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`PipelineEndpoint` is the entry point. A submission writes a `PipelineSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `PipelineWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PipelinePlannerAgent` to plan, then on each iteration asks it to decide. The decision routes to one of four specialist agents — `SparkAgent`, `BeamAgent`, `DbtAgent`, `SchemaAgent`. Every transition emits an event on `PipelineEntity`; `PipelineView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `RequestSimulator` drips sample pipeline requests for demo purposes; `StuckPipelineMonitor` ticks every 30 s to mark long-running `EXECUTING` pipelines as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → dispatch → test-gate → record → decide. The diagram condenses the four specialists into a single participant for readability; in code the workflow's `dispatchStep` switches on `StageDecision.engine` to call the matching agent.

## State machine

`PipelineEntity` has six states. `PLANNING` is the initial state. `EXECUTING` is the loop state — most events fire here without changing the status. A pipeline lands in one of four terminal states: `FINALISED` (happy path — all stages passed the test gate and the planner produced a definition), `FAILED` (planner exhausted its replan or failure budget), `HALTED` (operator pressed Halt), or `STUCK` (no progress for 5 minutes).

## Entity model

`PipelineEntity` is the system's source of truth; every transition writes one of nine event types. `SystemControlEntity` carries the operator halt flag. `RequestQueue` is the audit log of submissions. `PipelineView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `testGateStep` 30 s, `decideStep` 45 s, `composeDefinitionStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(engine, stageKind)` pair; the fourth becomes `Fail`.
- Idempotency: `(prompt, requestedBy)` over a 10 s window deduplicates `POST /api/pipelines`.
- Stuck detection: `StuckPipelineMonitor` every 30 s; pipelines `EXECUTING` for > 5 minutes are marked `STUCK`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
