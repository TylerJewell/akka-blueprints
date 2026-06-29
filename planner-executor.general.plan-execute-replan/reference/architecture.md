# Architecture — plan-execute-replan

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`GoalEndpoint` is the entry point. A submission writes a `GoalSubmitted` event to `GoalQueue` (event-sourced for audit). `GoalRequestConsumer` subscribes to that queue and starts an `ExecutionWorkflow` per submission. The workflow drives the planner-executor-replanner loop: first it asks `PlannerAgent` to produce an `ExecutionPlan`, then for each iteration it runs the guardrail check, calls `ExecutorAgent` to run the current step, records the `StepResult` as an `Observation`, calls `ReplannerAgent` to decide the next move, and evaluates the replanning decision for quality. Every transition emits an event on `GoalEntity`; `GoalView` projects those events into the read model the UI streams via SSE.

`RequestSimulator` drips sample goals every 90 s for demo purposes. `StuckGoalMonitor` ticks every 30 s to mark long-running `EXECUTING` goals as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its pause flag from the UI; the workflow polls the flag before every step.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor-replanner cycle — each iteration runs check-pause → guard → execute → observe → replan → eval → decide. The Replanner and quality evaluator run together in each loop iteration, making the evaluation overhead constant and predictable. The diagram labels the evaluator inline as a workflow step rather than a separate participant for readability; in code, `evalStep` calls `ReplanQualityEvaluator.score` directly — no agent call required.

## State machine

`GoalEntity` has six states. `PLANNING` is the initial state, lasting only until the `PlannerAgent` returns the first `ExecutionPlan`. `EXECUTING` is the loop state — most events fire here without changing the status. A goal lands in one of four terminal states: `CONCLUDED` (the Replanner produced a `Conclude` decision with sufficient evidence), `FAILED` (the revision budget or step failure budget was exhausted), `PAUSED` (the operator pressed Pause), or `STUCK` (no progress for 5 minutes).

## Entity model

`GoalEntity` is the source of truth; every transition writes one of eleven event types. `SystemControlEntity` carries the operator pause flag. `GoalQueue` is the audit log of submissions. `GoalView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `executeStep` 90 s, `replanStep` 45 s, `evalStep` 30 s, `concludeStep` 60 s.
- Revision budget: 3 `Revise` decisions; the fourth becomes `Fail`.
- Step failure budget: 2 consecutive failures on the same step before the Replanner is forced to `Revise` or `Fail`.
- Idempotency: `(goal, requestedBy)` over a 10 s window deduplicates `POST /api/goals`.
- Stuck detection: `StuckGoalMonitor` every 30 s; goals `EXECUTING` for > 5 minutes are marked `STUCK`.
- Pause poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
- Evaluator determinism: `ReplanQualityEvaluator.score` is a pure function — same inputs always produce the same `EvalRecord`, making the observation ledger replayable and auditable.
