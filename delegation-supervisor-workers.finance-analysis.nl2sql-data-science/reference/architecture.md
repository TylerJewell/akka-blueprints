# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A financial question enters through `AnalysisEndpoint` (`POST /api/analysis`) or is dripped by `QuestionSimulator` every 60 seconds. Either path writes a `QuestionSubmitted` event onto `QueryQueue`. `QueryRequestConsumer` subscribes to those events and starts one `AnalysisWorkflow` per submission, keyed by `jobId`.

The workflow is the supervisor. It first asks `QueryCoordinator` to translate the question into a SQL plan, then runs `guardrailStep` to inspect the SQL for destructive keywords. If the SQL passes, the workflow fans work out to `DataRetriever` and `StatisticsModeler` in parallel, then asks `QueryCoordinator` to synthesise. Every transition is written as a command to `AnalysisJobEntity`, whose events project into `AnalysisView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a completed job to score.

## Interaction sequence

The sequence diagram traces the primary journey: translate, guardrail check, parallel fan-out, join, synthesise, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results before calling the coordinator again. The guardrail check sits between translation and fan-out, ensuring no worker ever receives an unsafe SQL plan.

## State machine

`AnalysisJob` moves `TRANSLATING → RUNNING` when the SQL plan clears the guardrail, then to one of three terminals: `COMPLETE` (both workers returned and synthesis succeeded), `DEGRADED` (a worker timed out; synthesised from partial input), or `BLOCKED` (guardrail rejected the SQL). `COMPLETE` accepts one further `TranslationEvalScored` self-transition when `EvalSampler` records a fidelity score.

## Entity model

`QueryQueue` seeds one `AnalysisJob` per question. A job owns at most one `SqlPlan`, one `DataSet`, one `ModelResult`, and one `DataReport`. The view row mirrors the job with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
