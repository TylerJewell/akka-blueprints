# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A job enters through `JobEndpoint` (`POST /api/jobs`) or is dripped by `JobSimulator` every 60 seconds. Either path writes a `JobSubmitted` event onto `JobQueue`. `JobRequestConsumer` subscribes to those events and starts one `ParallelWorkflow` per submission, keyed by `jobId`.

The workflow is the supervisor of the fan-out. It asks `ParallelSupervisor` to partition the job (PARTITION task), then dispatches all worker calls simultaneously: `SectionWorker` calls for SECTIONING mode, `VoteWorker` calls for VOTING mode. All worker calls run concurrently — this is the core of the parallelization pattern. Once all results (or a timeout-triggered partial set) arrive, the workflow asks `ParallelSupervisor` to aggregate. Every lifecycle transition is written as a command to `JobEntity`, whose events project into `JobView`. `JobEndpoint` reads and streams the view; `EvalSampler` reads it to pick a completed job to score.

## Interaction sequence

The sequence diagram traces the primary journey: partition, parallel fan-out across N workers, join, aggregate, guardrail, persist. The `par` blocks show all workers running simultaneously regardless of mode. The guardrail step decides between the `AGGREGATED` and `BLOCKED` terminals. When a worker times out, the workflow routes to a partial path and the job enters `PARTIAL` without retrying.

## State machine

`Job` moves `PARTITIONING → IN_PROGRESS`, then to one of three terminals: `AGGREGATED` (guardrail passed), `PARTIAL` (at least one worker timed out and the guardrail accepted what arrived), or `BLOCKED` (guardrail rejected the aggregated output). `AGGREGATED` accepts one further `EvalScored` self-transition when `EvalSampler` records a quality score.

## Entity model

`JobQueue` seeds one `Job` per submission. A job holds at most one list of `SectionResult` items (SECTIONING) or `VoteResult` items (VOTING), and produces one `AggregatedOutput`. The view row mirrors the job with `Optional<T>` on every field that is null before its transition (Lesson 6).
