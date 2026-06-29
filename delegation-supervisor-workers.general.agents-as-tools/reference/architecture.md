# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A task request enters through `TaskEndpoint` (`POST /api/tasks`) or is dripped by `TaskSimulator` every 60 seconds. Either path writes a `TaskSubmitted` event onto `TaskQueue`. `TaskRequestConsumer` subscribes to those events and starts one `TaskWorkflow` per submission, keyed by `taskId`.

The workflow is the supervisor's host. It creates the `TaskRecord` in `QUEUED`, transitions it to `PROCESSING`, then invokes `TaskSupervisor` via a single `SUPERVISE` step. Inside the supervisor's own iteration loop, it calls `WriterAgent` via `writer_tool` and/or `DataAgent` via `data_tool` — these are agent-to-agent tool calls, not separate workflow steps. When the supervisor returns an `AssembledResult`, the workflow writes the terminal state to `TaskRecordEntity`, whose events project into `TaskView`. `TaskEndpoint` reads and streams the view; `QualitySampler` reads it to pick a completed task to score.

## Interaction sequence

The sequence diagram traces the primary journey: submit, create record, invoke supervisor, tool calls within supervisor, assemble result, persist terminal state. The supervisor's tool-call loop is the agents-as-tools core — the supervisor controls which tools to call and in what order, reducing the workflow to a single delegating step.

## State machine

`TaskRecord` moves `QUEUED → PROCESSING`, then to one of three terminals: `COMPLETED` (supervisor assembled a result), `REJECTED` (supervisor could not route the task to any tool), or `FAILED` (the supervise step timed out). `COMPLETED` accepts one further `QualityScored` self-transition when `QualitySampler` records a score.

## Entity model

`TaskQueue` seeds one `TaskRecord` per submission. A record owns at most one `DraftOutput` (from WriterAgent), one `DataOutput` (from DataAgent), and one `AssembledResult` (the supervisor's merged answer). The view row mirrors the record with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
