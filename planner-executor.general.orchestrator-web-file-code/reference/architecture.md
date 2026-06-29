# Architecture — orchestrator-web-file-code

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`TaskEndpoint` is the entry point. A submission writes a `TaskSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `TaskWorkflow` per submission. The workflow drives the planner-executor loop: it asks `OrchestratorAgent` to plan, then on each iteration asks it to decide. The decision routes to one of four specialist agents — `WebSurferAgent`, `FileSurferAgent`, `CoderAgent`, `TerminalAgent`. Every transition emits an event on `TaskEntity`; `TaskView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `RequestSimulator` drips sample tasks for demo purposes; `StuckTaskMonitor` ticks every 30 s to mark long-running `EXECUTING` tasks as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → dispatch → sanitize → record → auto-halt-eval → decide. The diagram condenses the four specialists into a single participant for readability; in code the workflow's `dispatchStep` switches on `DispatchDecision.specialist` to call the matching agent.

## State machine

`TaskEntity` has six states. `PLANNING` is the initial state. `EXECUTING` is the loop state — most events fire here without changing the status. A task lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (orchestrator exhausted its replan or failure budget), `HALTED` (operator pressed Halt or the auto-safety evaluator flagged an unsafe action), or `STUCK` (no progress for 5 minutes).

## Entity model

`TaskEntity` is the system's source of truth; every transition writes one of eleven event types. `SystemControlEntity` carries the operator halt flag. `RequestQueue` is the audit log of submissions. `TaskView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `completeStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(specialist, subtask)`; the fourth becomes `Fail`.
- Idempotency: `(prompt, requestedBy)` over a 10 s window deduplicates `POST /api/tasks`.
- Stuck detection: `StuckTaskMonitor` every 30 s; tasks `EXECUTING` for > 5 minutes are marked `STUCK`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
