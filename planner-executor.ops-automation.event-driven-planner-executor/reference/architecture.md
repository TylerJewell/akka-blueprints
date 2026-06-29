# Architecture — event-driven-planner-executor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`JobEndpoint` is the entry point. A submission writes a `JobSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `JobWorkflow` per submission. The workflow drives the planner-executor loop: it asks `OrchestratorAgent` to plan, then on each iteration asks it to decide. The decision routes to one of four executor agents — `HttpCallerAgent`, `QueuePublisherAgent`, `DbQueryAgent`, `ScriptRunnerAgent`. Every transition emits an event on `JobEntity`; `JobView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `JobSimulator` drips sample jobs for demo purposes; `StuckJobMonitor` ticks every 30 s to mark long-running `EXECUTING` jobs as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → dispatch → sanitize → record → decide. The diagram condenses the four executors into a single participant for readability; in code the workflow's `dispatchStep` switches on `DispatchDecision.executor` to call the matching agent.

## State machine

`JobEntity` has six states. `PLANNING` is the initial state. `EXECUTING` is the loop state — most events fire here without changing the status. A job lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (orchestrator exhausted its replan or failure budget), `HALTED` (operator pressed Halt), or `STUCK` (no progress for 5 minutes).

## Entity model

`JobEntity` is the system's source of truth; every transition writes one of nine event types. `SystemControlEntity` carries the operator halt flag. `RequestQueue` is the audit log of submissions. `JobView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `composeReportStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(executor, step)`; the fourth becomes `Fail`.
- Idempotency: `(prompt, requestedBy)` over a 10 s window deduplicates `POST /api/jobs`.
- Stuck detection: `StuckJobMonitor` every 30 s; jobs `EXECUTING` for > 5 minutes are marked `STUCK`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
