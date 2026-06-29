# Architecture — sk-process

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`JobQueueEndpoint` is the entry point. A submitted job is logged as a `JobSubmitted` event on `JobQueue` (event-sourced for audit). `JobRequestConsumer` subscribes to that queue, creates a `JobEntity`, and starts a `ProcessOrchestrationWorkflow` keyed by the job id. That workflow is the coordinator: it runs the job through five stages, writing each stage's result onto the shared `JobEntity` before the next begins.

The three desks under the pipeline each run a different internal coordination capability over the same shared workspace:

- **Planning desk — delegation.** `planStep` and `detailStep` run `ProcessCoordinator` and `StepPlanner` sequentially; the coordinator sets the objective and step list, the planner refines each step into an executor-ready spec and seeds the board with one `StepEntity` per step.
- **Execution desk — a team over a shared list.** `executionStep` writes the step board and then waits. The executor roster (`executor-1`, `executor-2`, `executor-3`) runs independently: `Bootstrap` starts one `ExecutorWorkflow` per executor, each polling `StepBoardView`, claiming an open step atomically, running the `StepExecutor` agent, and marking the step done. An operator can pause the workflow from the dashboard at any time during this stage.
- **Validation desk — moderation.** `validationStep` runs one `QualityChecker` instance per criterion (`format`, `completeness`, `policy`), each evaluating the completed batch and writing a note; the deterministic `QualityRule` turns the panel's notes into a pass-or-retry verdict.

`StepEvalConsumer` subscribes to the job's stage events and records a non-blocking quality signal after each stage result. Two TimedActions sit alongside: `JobSimulator` drips sample process templates; `StuckStepMonitor` releases stranded step claims. `AppEndpoint` serves the embedded UI and the metadata the tabs read.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → plan → detail-plan (seed the board) → executors claim and fill steps → assemble the batch → the panel scores → a passing verdict finalises the job. Two `Note over` blocks mark the asynchronous handoffs: the executor loops are already polling the board before any step exists, and the execution stage waits by polling rather than blocking. A third marks where the before-agent-response guardrail vets the final `JobSummary`.

## State machine

`JobEntity` is the spine of the pipeline. It moves `SUBMITTED → PLANNED → EXECUTING → VALIDATING`, and then either `APPROVED → COMPLETED` on a passing review or back to `EXECUTING` for one bounded retry round on a `RETRY` verdict. Two self-transitions and a branch matter: `recordFinaliseBlock` keeps the job `APPROVED` when the output guardrail refuses the summary (nothing is finalised), `recordAuditReview` keeps the job `COMPLETED` when a post-completion audit review lands, and `pauseJob` / `resumeJob` form a round-trip from and back to `EXECUTING` without replanning any steps.

## Entity model

`JobEntity` is the shared workspace and the source of truth for a job; every stage writes one of twelve event types, and `JobBoardView` is its read-side projection. `StepEntity` is the execution team's coordination point — one per step, with an atomic `claim` that the single-writer guarantee resolves to exactly one winner under contention; `StepBoardView` is the shared list the executors poll. `JobQueue` is the audit log of submissions. `StepEvalConsumer` reads job events to produce quality signals; it writes its signal back through a command, not by mutating state directly.

## Concurrency, pause, and timeouts

- The top-level pipeline is sequential delegation; the execution stage hands off to an independent team and waits on the board.
- Atomic claim through `StepEntity`'s single writer is the execution-team primitive — no lock, no external queue.
- Per-step timeouts on the agent-calling steps: `planStep` 60 s, `detailStep` 60 s, `validationStep` 120 s (it fans out three checker calls), `finaliseStep` 60 s, and `ExecutorWorkflow.executeStep` 120 s.
- Pause/resume is a checkpoint at a step boundary: `executionStep` emits `JobPaused` and suspends; executor workflows detect the paused state and idle rather than claiming new steps; `resumeJob` re-enters the same poll-until-done loop.
- The execution wait and idle executors are paused workflows with short resume timers, not busy loops.
- The retry loop runs at most once, so the pipeline always terminates.
- `StuckStepMonitor` releases a step claimed-but-idle for more than three minutes so a failed executor does not strand the board.
- `QualityRule` and `StepEvaluator` are deterministic pure functions (no LLM call), so the quality verdict and the step signals are reproducible.
