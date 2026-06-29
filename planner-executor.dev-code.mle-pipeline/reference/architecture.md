# Architecture — mle-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`PipelineEndpoint` is the entry point. A submission writes a `RunSubmitted` event to `RunQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `PipelineWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to stage the pipeline, then on each iteration asks it to decide the next stage. The decision routes to one of four specialist agents — `DataProfilerAgent`, `FeatureEngineerAgent`, `ModelTrainerAgent`, `ModelEvaluatorAgent`. Every transition emits an event on `PipelineRunEntity`; `PipelineRunView` projects those events into the read model the UI streams via SSE.

Three TimedActions run alongside: `RunSimulator` drips sample runs for demo purposes; `StaleRunMonitor` ticks every 60 s to mark long-running `EXECUTING` runs as `STUCK`; `DriftWatchMonitor` ticks every 5 minutes to compare completed runs' metrics against baselines and emit `DriftAlertRaised` when thresholds are exceeded.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every stage dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → gate → dispatch → record → drift-check → decide. The diagram condenses the four specialists into a single participant for readability; in code the workflow's `dispatchStep` switches on `StageDispatch.specialist` to call the matching agent.

## State machine

`PipelineRunEntity` has seven states. `PLANNING` is the initial state. `EXECUTING` is the loop state — most events fire here without changing status, including `DriftAlertRaised` which can fire even after a run is `COMPLETED`. A run lands in one of five terminal states: `COMPLETED` (happy path, gate passed), `GATE_FAILED` (gate budget exhausted), `FAILED` (planner exhausted its replan budget), `HALTED` (operator pressed Halt), or `STUCK` (no progress for 10 minutes).

## Entity model

`PipelineRunEntity` is the system's source of truth; every transition writes one of twelve event types, including `DriftAlertRaised` which appends to the run's `driftAlerts` list without changing status. `SystemControlEntity` carries the operator halt flag. `RunQueue` is the audit log of submissions. `PipelineRunView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `promoteStep` 60 s.
- Replan budget: 3 consecutive `Replan` outputs; the fourth becomes `Fail`.
- Gate budget: 2 consecutive `StageGateFailed` records; the third triggers `Fail`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
- Idempotency: `(datasetRef, objective)` over a 10 s window deduplicates `POST /api/runs`.
- Stuck detection: `StaleRunMonitor` every 60 s; runs `EXECUTING` for > 10 minutes are marked `STUCK`.
- Drift evaluation determinism: `DriftEvaluator.check` is pure — same `(MetricBundle, baseline)` always yields the same alerts.
