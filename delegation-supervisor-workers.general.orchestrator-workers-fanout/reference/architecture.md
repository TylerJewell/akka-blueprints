# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A task enters through `TaskEndpoint` (`POST /api/tasks`) or is dripped by `TaskSimulator` every 60 seconds. Either path writes a `TaskSubmitted` event onto `TaskQueue`. `TaskRequestConsumer` subscribes to those events and starts one `TaskWorkflow` per submission, keyed by `requestId`.

The workflow is the supervisor. It asks `TaskOrchestrator` to decompose the task description into an `ExecutionPlan`, runs a `planGuardrailStep` to validate that plan, then fans the work out to `WriterWorker` and `AnalyzerWorker` in parallel. After both workers return, it asks `TaskOrchestrator` to synthesise a `CompositeResult`. Every transition is written as a command to `TaskRequestEntity`, whose events project into `TaskView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a completed task to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, plan validation, parallel fan-out, join, synthesise, persist. The plan-validation decision appears before the `par` block — the key distinction of this blueprint versus the research-intel canonical is that the guardrail fires before workers are dispatched, not after synthesis. The `par` block is the delegation-supervisor-workers core; both workers run concurrently and the workflow joins their results.

## State machine

`TaskRequest` moves `PLANNING → IN_PROGRESS`, then to one of three terminals: `COMPLETED` (synthesis succeeded), `DEGRADED` (a worker timed out), or `BLOCKED` (plan guardrail failed). `PLANNING` can also transition directly to `BLOCKED` if the plan fails validation before any worker runs. `COMPLETED` accepts one further `TaskEvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`TaskQueue` seeds one `TaskRequest` per task description. A task request owns at most one `ExecutionPlan`, one `WriterOutput`, one `AnalyzerOutput`, and one `CompositeResult`. The view row mirrors the request with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
