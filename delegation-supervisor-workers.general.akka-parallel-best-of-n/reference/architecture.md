# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A translation job enters through `TranslationEndpoint` (`POST /api/translation`) or is dripped by `JobSimulator` every 60 seconds. Either path writes a `JobSubmitted` event onto `JobQueue`. `JobQueueConsumer` subscribes to those events and starts one `TranslationWorkflow` per submission, keyed by `jobId`.

The workflow is the supervisor. It asks `TranslationSupervisor` to produce a `VariantPlan`, then fans the work out to three `TranslationWorker` branches concurrently — one per register style. All three results are collected and passed back to `TranslationSupervisor` for selection. Every lifecycle transition is written as a command to `TranslationJobEntity`, whose events project into `TranslationView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a completed job to score.

## Interaction sequence

The sequence diagram traces the primary journey: plan variants, parallel fan-out to three workers, join, select best, guardrail, persist. The `par` block is the delegation-supervisor-workers core — all three workers run concurrently and the workflow joins their results before calling the supervisor a second time. The guardrail decides between the `SELECTED` and `BLOCKED` terminals.

## State machine

`TranslationJob` moves `PENDING → IN_PROGRESS`, then to one of three terminals: `SELECTED` (guardrail passed), `PARTIAL` (one or more branches timed out but selection proceeded from the available variants), or `BLOCKED` (guardrail failed). `SELECTED` accepts one further `SelectionEvalScored` self-transition when `EvalSampler` records a quality score.

## Entity model

`JobQueue` seeds one `TranslationJob` per submission. A job collects up to three `TranslationVariant` items (one per branch) and produces at most one `SelectionResult`. The view row mirrors the job with `Optional<T>` on every field that is null before its transition (Lesson 6).
