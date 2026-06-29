# Architecture — bq-pipeline-builder

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`PipelineEndpoint` is the entry point. A submission writes a `PipelineSubmitted` event to `PipelineQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `PipelineWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to plan, then on each iteration asks it to decide which build step runs next. The decision routes to one of four specialist executors — `SchemaAnalystAgent`, `SqlComposerAgent`, `DataformModelerAgent`, `ValidatorAgent`. Every transition emits an event on `PipelineEntity`; `PipelineView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `PipelineSimulator` drips sample pipeline requests every 90 seconds for demo purposes; `StalePipelineMonitor` ticks every 30 s to mark long-running `BUILDING` pipelines as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → dispatch → ci-gate → record → decide. The diagram condenses the four specialist executors into a single participant for readability; in code the workflow's `dispatchStep` switches on `BuildStepDecision.executor` to call the matching agent.

## State machine

`PipelineEntity` has six states. `PLANNING` is the initial state. `BUILDING` is the loop state — most events fire here without changing the status. A pipeline lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (planner exhausted its replan or failure budget), `HALTED` (operator pressed Halt), or `STUCK` (no progress for 5 minutes).

## Entity model

`PipelineEntity` is the system's source of truth; every transition writes one of nine event types. `SystemControlEntity` carries the operator halt flag. `PipelineQueue` is the audit log of submissions. `PipelineView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `ciGateStep` 30 s, `decideStep` 45 s, `completeStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(executor, step)`; the fourth becomes `Fail`.
- Idempotency: `(description, requestedBy)` over a 10 s window deduplicates `POST /api/pipelines`.
- Stuck detection: `StalePipelineMonitor` every 30 s; pipelines `BUILDING` for > 5 minutes are marked `STUCK`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
- CI gate determinism: `ValidationSuite.run` is pure — it reads only the step output text and the fixture schemas loaded at startup. The same input always yields the same `ValidationReport`, keeping `StepEntry` events deterministic and replayable across event-sourced entity snapshots.
