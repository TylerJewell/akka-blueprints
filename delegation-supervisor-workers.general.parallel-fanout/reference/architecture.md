# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A job enters through `TaskEndpoint` (`POST /api/tasks`) or is dripped by `JobSimulator` every 60 seconds. Either path writes a `JobSubmitted` event onto `JobQueue`. `JobRequestConsumer` subscribes to those events and starts one `TaskExecutionWorkflow` per submission, keyed by `jobId`.

The workflow is the supervisor. It asks `TaskCoordinator` to decompose the payload, fans the work out to `SubtaskWorkerA` and `SubtaskWorkerB` in parallel, then asks `TaskCoordinator` to consolidate. Every transition is written as a command to `TaskJobEntity`, whose events project into `TaskJobView`. The endpoint reads and streams the view; `QualitySampler` reads it to pick a job to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out, join, consolidate, validate, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The validation step decides between the `CONSOLIDATED` and `REJECTED` terminals.

## State machine

`TaskJob` moves `QUEUED → PROCESSING`, then to one of three terminals: `CONSOLIDATED` (validation passed), `DEGRADED` (a worker timed out), or `REJECTED` (validation failed). `CONSOLIDATED` accepts one further `QualityScored` self-transition when `QualitySampler` records a score.

## Entity model

`JobQueue` seeds one `TaskJob` per submission. A job owns at most one `StructuralOutput`, one `ContextualOutput`, and one `ConsolidatedResult`. The view row mirrors the job with `Optional<T>` on every field that is null before its transition (Lesson 6).
