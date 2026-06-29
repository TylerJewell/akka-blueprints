# Architecture — shared-task-team

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SharedTaskEndpoint` is the entry point. A submitted goal is logged as a `GoalSubmitted` event on `IntakeQueue` (event-sourced for audit). `GoalRequestConsumer` subscribes to that queue, creates a `GoalEntity`, and starts a `PlanningWorkflow`. The workflow runs the `GoalPlanner` agent to decompose the brief, then writes one `TaskEntity` per task onto the board (each `OPEN`). Every task transition projects into `TaskBoardView` — the shared list.

The worker side runs in parallel and independently. `Bootstrap` starts one `WorkerWorkflow` per worker id (`worker-1`, `worker-2`, `worker-3`). Each worker loop polls `TaskBoardView`, claims an eligible task on its `TaskEntity`, runs the `WorkerAgent`, runs the artifact through the quality check, and marks the task done or blocked. When a worker is blocked on coordination it posts to a peer's `WorkerMailbox`. `SystemControl` (a key-value entity) holds the operator halt flag that both the poll loop and the response guardrail read. Two TimedActions sit alongside: `GoalSimulator` drips sample briefs; `StuckTaskMonitor` releases stranded claims.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → plan → tasks on the board → a worker claims, produces an artifact, passes the quality check, and the task goes `DONE`. The `Note over WorkerAgent` block marks where the before-agent-response guardrail vets every synthesized response before it is committed. The worker loops are already polling before any task exists, so the only thing the plan adds is rows on the board for them to find.

## State machine

`TaskEntity` is the heart of the pattern. `OPEN` is the initial state. `claim` is atomic — only an `OPEN` task can be claimed, and the entity's single-writer guarantee means exactly one worker wins a contested task. From `CLAIMED` the worker `start`s into `IN_PROGRESS`, submits an artifact into `IN_REVIEW`, and the quality check decides: `DONE` on pass, back to `IN_PROGRESS` on a retryable failure, `BLOCKED` once retries are exhausted. A peer request or a guardrail refusal moves the task to `BLOCKED` directly. `StuckTaskMonitor` returns a claimed-but-idle task to `OPEN` for liveness, and a peer reply (or operator reopen) returns a blocked task to `OPEN`.

## Entity model

`TaskEntity` is the source of truth for each unit of work; every transition writes one of nine event types, and `TaskBoardView` is the only read-side projection — the worker loops never query the entity directly, they read the board. `GoalEntity` tracks the goal lifecycle and owns the set of task ids, and it receives the on-decision eval result after completion. `IntakeQueue` is the audit log of submissions. `WorkerMailbox` (one per worker) holds the cross-worker coordination messages.

## Concurrency & timeouts

- Atomic claim through `TaskEntity`'s single writer is the coordination primitive — no lock, no external queue.
- Per-step timeout: 90 s on the agent-calling steps (`PlanningWorkflow.decomposeStep`, `WorkerWorkflow.workStep`).
- Idle workers are paused workflows with a 5 s resume timer, not busy loops.
- A task is eligible only when every `dependsOn` title resolves to a `DONE` task — checked client-side because the view exposes no enum-status filter.
- `StuckTaskMonitor` releases a task claimed-but-idle for more than two minutes so a failed worker does not strand work.
- The quality check (`QualityChecker`) is deterministic, so a given artifact always yields the same verdict; the Maven `test` phase enforces the same gate at build time.
- The on-decision eval runs after goal completion, non-blocking; it does not delay the `COMPLETED` transition.
