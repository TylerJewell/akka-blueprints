# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A research request enters through `ReportEndpoint` (`POST /api/reports`) or is dripped by `RequestSimulator` every 60 seconds. Either path writes a `ResearchSubmitted` event onto `RequestQueue`. `RequestConsumer` subscribes to those events and starts one `ReportWorkflow` per submission, keyed by `reportId`.

The workflow is the supervisor. It asks `ReportCoordinator` to produce a `ResearchPlan`, fans the work out to `EquityAnalyst` and `MarketScout` in parallel, then asks `ReportCoordinator` to synthesise. Every transition is written as a command to `ResearchReportEntity`, whose events project into `ReportView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a report to score.

## Interaction sequence

The sequence diagram traces the primary journey: plan, parallel fan-out, join, synthesise, sanitize, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The sanitizer step decides between the `PUBLISHED` and `BLOCKED` terminals.

## State machine

`ResearchReport` moves `PLANNING → IN_PROGRESS`, then to one of three terminals: `PUBLISHED` (sanitizer passed), `DEGRADED` (a worker timed out), or `BLOCKED` (sanitizer failed). `PUBLISHED` accepts one further `ReportEvalScored` self-transition when `EvalSampler` records a numeric-claim quality score.

## Entity model

`RequestQueue` seeds one `ResearchReport` per request. A report owns at most one `FundamentalsBundle`, one `SentimentBundle`, and one `SynthesisedReport`. The view row mirrors the report with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
