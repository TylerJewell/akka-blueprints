# Architecture — field-service-dispatcher

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ScheduleEndpoint` is the entry point. A work order submission writes a `WorkOrderSubmitted` event to `WorkOrderQueue` (event-sourced for audit). A `WorkOrderConsumer` subscribes to that queue and starts a `ScheduleWorkflow` per submission. The workflow drives the planner-executor loop: it asks `DispatcherAgent` to plan, then on each iteration asks it to decide. The decision routes to one of two specialist agents — `RouteOptimizerAgent` or `AvailabilityAgent`. Every transition emits an event on `ScheduleEntity`; `ScheduleView` projects those events into the read model the UI streams via SSE.

Three TimedActions run alongside: `WorkOrderSimulator` drips sample work orders for demo purposes; `StalledScheduleMonitor` ticks every 30 s to mark long-running `DISPATCHING` schedules as `STALLED`; `FairnessMonitor` ticks every 5 minutes to evaluate assignment distribution across completed schedules and record fairness alerts.

`OperatorControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its pause flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-pause → propose → guardrail → dispatch → sanitize → record → fairness-check → decide. The diagram condenses both specialists into a single participant for readability; in code the workflow's `dispatchStep` switches on `AssignmentDecision.specialist` to call the matching agent.

## State machine

`ScheduleEntity` has six states. `PLANNING` is the initial state set when the entity is created. `DISPATCHING` is the loop state — most events fire here without changing the status. A schedule lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (dispatcher exhausted its replan or failure budget), `PAUSED` (operator pressed Pause), or `STALLED` (no progress for 5 minutes). `FairnessAlertRecorded` events fire in `DISPATCHING` or after `COMPLETED` without changing the status; they are advisory.

## Entity model

`ScheduleEntity` is the system's source of truth; every transition writes one of eleven event types. `OperatorControlEntity` carries the operator pause flag. `WorkOrderQueue` is the audit log of submissions. `ScheduleView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `completeStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(specialist, assignment)`; the fourth becomes `Fail`.
- Idempotency: `(description, serviceAddress, requiredSkill)` over a 10 s window deduplicates `POST /api/schedules`.
- Stalled detection: `StalledScheduleMonitor` every 30 s; schedules `DISPATCHING` for > 5 minutes are marked `STALLED`.
- Pause poll: synchronous read of `OperatorControlEntity` at the top of every loop iteration; no caching.
- Fairness monitor: non-blocking; alerts are advisory records on the entity; does not interrupt in-progress workflows.
