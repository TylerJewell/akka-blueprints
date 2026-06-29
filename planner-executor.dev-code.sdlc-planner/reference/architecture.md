# Architecture — sdlc-task-planner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`PlanEndpoint` is the entry point. A submission writes a `PlanSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `PlanWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to decompose the request, then on each iteration asks it to decide. The decision routes to one of four specialist agents — `AnalystAgent`, `ArchitectAgent`, `CoderAgent`, `ReviewerAgent`. Every transition emits an event on `PlanEntity`; `PlanView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `RequestSimulator` drips sample SDLC requests for demo purposes; `StalePlanMonitor` ticks every 30 s to mark long-running `EXECUTING` plans as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → dispatch → sanitize → record → auto-halt-eval → decide. The diagram condenses the four specialists into a single participant for readability; in code the workflow's `dispatchStep` switches on `DispatchDecision.specialist` to call the matching agent.

## State machine

`PlanEntity` has six states. `PLANNING` is the initial state. `EXECUTING` is the loop state — most events fire here without changing the status. A plan lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (planner exhausted its replan or failure budget), `HALTED` (operator pressed Halt or the auto-safety evaluator flagged an unsafe output), or `STUCK` (no progress for 5 minutes).

## Entity model

`PlanEntity` is the system's source of truth; every transition writes one of eleven event types. `SystemControlEntity` carries the operator halt flag. `RequestQueue` is the audit log of submissions. `PlanView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `decomposeStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `completeStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(specialist, subtask)`; the fourth becomes `Fail`.
- Idempotency: `(prompt, requestedBy)` over a 10 s window deduplicates `POST /api/plans`.
- Stuck detection: `StalePlanMonitor` every 30 s; plans `EXECUTING` for > 5 minutes are marked `STUCK`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
