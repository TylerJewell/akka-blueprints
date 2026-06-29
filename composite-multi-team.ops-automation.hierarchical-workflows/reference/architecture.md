# Architecture — hierarchical-workflow-automation

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`WorkflowEndpoint` is the entry point. A submitted operations request is logged as a `RequestSubmitted` event on `RequestQueue` (event-sourced for audit). `RequestConsumer` subscribes to that queue, creates a `WorkflowEntity`, and starts a `WorkflowInstance` keyed by the request id. That workflow is the orchestrator: it runs the request through five stages, writing each stage's result onto the shared `WorkflowEntity` before the next begins.

The three specialist teams under the pipeline each run a different internal coordination capability over the same shared workspace:

- **Discovery team — delegation.** `discoveryStep` runs `DiscoveryLead` to plan scan targets, fans out one `SystemScanner` instance per target (each writing a scan result into the workspace through `SystemTools`), then runs `DiscoveryLead` again to synthesise the results into a discovery summary.
- **Execution team — a team over a shared list.** `executionStep` runs `ExecutionLead` to plan tasks, writes one `TaskEntity` per task onto the board, and then waits. The executor roster (`executor-1`, `executor-2`) runs independently: `Bootstrap` starts one `ExecutorWorkflow` per executor, each polling `TaskBoardView`, claiming an open task atomically, running the `TaskExecutor` agent, and marking the task done.
- **Validation team — moderation.** `validationStep` runs one `Validator` instance per axis (`correctness`, `safety`, `compliance`), each scoring the execution summary and writing a note; the deterministic `AggregationRule` turns the panel's notes into a pass-or-retry verdict.

`AuditEvalConsumer` subscribes to the workflow's stage events and records a non-blocking quality eval after each stage result. Two TimedActions sit alongside: `RequestSimulator` drips sample requests; `StuckTaskMonitor` releases stranded task claims. `AppEndpoint` serves the embedded UI and the metadata the tabs read.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → plan → discovery (delegate and synthesise) → plan tasks → executors claim and complete the board → assemble the execution summary → the panel validates → a passing verdict assembles and delivers the operations report. Two `Note over` blocks mark the asynchronous handoffs: the executor loops are already polling the board before any task exists, and the execution stage waits by polling rather than blocking. A third marks where the before-agent-response guardrail vets the final report.

## State machine

`WorkflowEntity` is the spine of the pipeline. It moves `SUBMITTED → PLANNED → DISCOVERING → DISCOVERED → EXECUTING → EXECUTED → VALIDATING`, and then either `VALIDATED → COMPLETED` on a passing validation or back to `EXECUTING` for one bounded retry round on a `RETRY` verdict. Two self-transitions matter: `recordReportBlock` keeps the workflow `VALIDATED` when the output guardrail refuses the report (nothing is completed), and `recordComplianceReview` keeps the workflow `COMPLETED` when a post-execution compliance review lands — the reviewer is on the loop, not gating it.

## Entity model

`WorkflowEntity` is the shared workspace and the source of truth for a request; every stage writes one of ten event types, and `WorkflowBoardView` is its read-side projection. `TaskEntity` is the execution team's coordination point — one per task, with an atomic `claim` that the single-writer guarantee resolves to exactly one winner under contention; `TaskBoardView` is the shared list the executors poll. `RequestQueue` is the audit log of submissions. `AuditEvalConsumer` reads workflow events to score stages; it writes its eval back through a command, not by mutating state directly.

## Concurrency & timeouts

- The top-level pipeline is sequential delegation; the execution stage hands off to an independent team and waits on the board.
- Atomic claim through `TaskEntity`'s single writer is the execution-team primitive — no lock, no external queue.
- Per-step timeouts on the agent-calling steps: `planStep` 60 s, `discoveryStep` 120 s (it fans out several scanner calls), `validationStep` 90 s, `reportStep` 60 s, and `ExecutorWorkflow.executeStep` 90 s.
- The execution wait and idle executors are paused workflows with a 5 s resume timer, not busy loops.
- The retry loop runs at most once, so the pipeline always terminates.
- `StuckTaskMonitor` releases a task claimed-but-idle for more than two minutes so a failed executor does not strand the board.
- `AggregationRule` and `ResultEvaluator` are deterministic pure functions (no LLM call), so the validation verdict and the stage scores are reproducible.
