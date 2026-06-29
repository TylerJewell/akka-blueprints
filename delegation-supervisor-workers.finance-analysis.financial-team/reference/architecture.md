# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A financial query enters through `FinancialEndpoint` (`POST /api/financial`) or is dripped by `QuerySimulator` every 60 seconds. Either path writes a `QuerySubmitted` event onto `QueryQueue`. `QueryRequestConsumer` subscribes to those events and starts one `FinancialWorkflow` per submission, keyed by `reportId`.

The workflow is the supervisor. It asks `FinancialCoordinator` to decompose the query into three work assignments, fans the work out to `MarketResearcher`, `PortfolioPlanner`, and `ReportDrafter` in parallel, then asks `FinancialCoordinator` to synthesise. Every transition is written as a command to `FinancialReportEntity`, whose events project into `FinancialView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a report to score.

## Interaction sequence

The sequence diagram traces the primary journey: assign, parallel fan-out across three workers, join, synthesise, sanitize, guardrail, persist. The `par` block is the delegation-supervisor-workers core — all three workers run concurrently and the workflow joins their results before calling the coordinator for synthesis. The sanitizer and guardrail decide between the `PUBLISHED` and `BLOCKED` terminals.

## State machine

`FinancialReport` moves `DRAFTING → IN_REVIEW`, then to one of three terminals: `PUBLISHED` (sanitizer + guardrail passed), `DEGRADED` (a worker timed out), or `BLOCKED` (guardrail failed). `PUBLISHED` accepts one further `ReportEvalScored` self-transition when `EvalSampler` records a quality score.

## Entity model

`QueryQueue` seeds one `FinancialReport` per query. A report owns at most one `MarketDataBundle`, one `PlanningAssessment`, one `ReportNarrative`, and one `ConsolidatedReport`. The view row mirrors the report with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
