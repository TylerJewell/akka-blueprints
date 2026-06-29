# Architecture — data-processing-planner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative shown beside each diagram.

## Component graph

`PipelineEndpoint` is the entry point. A submission writes a `PipelineSubmitted` event to `PipelineQueue` (event-sourced for audit). A `PipelineRequestConsumer` subscribes to that queue and starts a `PipelineWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to plan, then on each iteration asks it to decide. The decision routes to `JobExecutorAgent`, which simulates the specified job engine (Glue crawler, EMR step, Spark submit, or S3 copy). Every transition emits an event on `PipelineEntity`; `PipelineView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `PipelineSimulator` drips sample requests for demo purposes; `StuckPipelineMonitor` ticks every 30 s to mark long-running `RUNNING` pipeline runs as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → execute → sanitize → record → decide. The four engine types are handled by a single `JobExecutorAgent`; the workflow's `executeStep` passes the engine enum in the dispatch so the agent selects the right fixture set. The diagram condenses this into a single "JobExecutorAgent" participant for readability.

## State machine

`PipelineEntity` has six states. `PLANNING` is the initial state. `RUNNING` is the loop state — most events fire here without changing the status. A pipeline run lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (planner exhausted its replan or failure budget), `HALTED` (operator cancelled the run), or `STUCK` (no progress for 8 minutes).

## Entity model

`PipelineEntity` is the system's source of truth; every transition writes one of nine event types. `SystemControlEntity` carries the operator halt flag — a single global flag that gates all concurrent pipeline runs. `PipelineQueue` is the audit log of submissions, kept separately so the queue can replay independently of any individual pipeline run's lifecycle. `PipelineView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency and timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `executeStep` 180 s, `decideStep` 45 s, `composeOutputStep` 60 s.
- Replan budget: 2 consecutive `RevisePlan` outputs; the third becomes `Fail`.
- Failure budget: 2 consecutive `ContinueDispatch` outputs on the same `(engine, stepSpec)`; the third becomes `Fail`.
- Idempotency: `(description, targetDataset, requestedBy)` over a 10 s window deduplicates `POST /api/pipelines`.
- Stuck detection: `StuckPipelineMonitor` every 30 s; runs `RUNNING` for > 8 minutes are marked `STUCK`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching. The halt flag applies across all concurrent pipeline runs — halting is a system-level gate, not a per-run control.
