# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A topic enters through `ReportEndpoint` (`POST /api/reports`) or is dripped by `TopicSimulator` every 60 seconds. Either path writes a `TopicSubmitted` event onto `TopicQueue`. `ReportRequestConsumer` subscribes to those events and starts one `ReportWorkflow` per submission, keyed by `reportId`.

The workflow is the supervisor. It asks `ReportCoordinator` to decompose the topic into a writing plan, fans the work out to `SourceResearcher` and `DraftWriter` in parallel, then asks `ReportCoordinator` to assemble the final report. Every transition is written as a command to `ReportEntity`, whose events project into `ReportView`. The endpoint reads and streams the view; `QualitySampler` reads it to pick a report to score.

## Interaction sequence

The sequence diagram traces the primary journey: plan, parallel fan-out, join, assemble, guardrail, persist. The `par` block is the delegation-supervisor-workers core -- both workers run concurrently and the workflow joins their results. The guardrail decides between the `ASSEMBLED` and `BLOCKED` terminals.

## State machine

`Report` moves `PLANNING -> IN_PROGRESS`, then to one of three terminals: `ASSEMBLED` (guardrail passed), `DEGRADED` (a worker timed out), or `BLOCKED` (guardrail failed). `ASSEMBLED` accepts one further `ReportScored` self-transition when `QualitySampler` records a score.

## Entity model

`TopicQueue` seeds one `Report` per topic. A report owns at most one `SourceBundle`, one `DraftSection`, and one `AssembledReport`. The view row mirrors the report with `Optional<T>` on every field that is null before its transition (Lesson 6).
