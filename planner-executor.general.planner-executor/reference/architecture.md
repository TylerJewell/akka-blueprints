# Architecture — planner-executor-general-planner-executor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative shown beside each diagram.

## Component graph

`PlanEndpoint` is the entry point. A submission writes a `GoalSubmitted` event to `GoalQueue` (event-sourced for audit). `GoalRequestConsumer` subscribes to that queue and starts a `PlanWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to create an initial plan, then on each iteration asks it to decide the next action. The decision routes to `ExecutorAgent`, which runs the step and returns a `StepResult`. Every transition emits an event on `PlanEntity`; `PlanView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `GoalSimulator` drips sample goals for demo purposes; `StuckPlanMonitor` ticks every 30 s to mark long-running `EXECUTING` plans as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every step dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → execute → record → auto-halt-eval → decide. The Planner and Executor are distinct participants with distinct roles; the workflow routes between them rather than calling them in parallel.

## State machine

`PlanEntity` has six states. `PLANNING` is the initial state — entered when the workflow starts and the Planner has not yet produced a ledger. `EXECUTING` is the loop state; most events fire here without changing the status. A plan lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (planner exhausted its replan or failure budget), `HALTED` (operator pressed Halt or the auto-safety evaluator flagged an unsafe result), or `STUCK` (no progress for 5 minutes).

## Entity model

`PlanEntity` is the source of truth; every transition writes one of eleven event types. `SystemControlEntity` carries the operator halt flag. `GoalQueue` is the audit log of submissions. `PlanView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `createPlanStep` 60 s, `proposeStep` 45 s, `executeStep` 120 s, `decideStep` 45 s, `composeOutcomeStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same step text; the fourth becomes `Fail`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
- Idempotency: `(goal, requestedBy)` over a 10 s window deduplicates `POST /api/plans`.
- Stuck detection: `StuckPlanMonitor` every 30 s; plans `EXECUTING` for > 5 minutes are marked `STUCK`.
- Guardrail determinism: `StepGuardrail.vet` is pure; the same `StepDecision` always yields the same verdict.
