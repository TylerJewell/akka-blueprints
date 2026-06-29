# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A query enters through `ReportEndpoint` (`POST /api/reports`) or is dripped by `TickerSimulator` every 60 seconds. Either path writes a `QuerySubmitted` event onto `RequestQueue`. `ReportRequestConsumer` subscribes to those events and starts one `ReportWorkflow` per submission, keyed by `reportId`.

The workflow is the supervisor. It first asks `ReportCoordinator` to decompose the query into a `QueryPlan`, then tasks `TickerResolver` to normalise the input to a canonical ticker. With the resolved ticker, the workflow fans out to `PriceAnalyst`, `NewsAnalyst`, and `FundamentalsAnalyst` in parallel. The coordinator then merges all four worker outputs into an `IntegratedReport`. Every lifecycle transition is written as a command to `StockReportEntity`, whose events project into `StockReportView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a published report to score.

## Interaction sequence

The sequence diagram traces the primary journey: plan query, resolve ticker, parallel three-way fan-out, join, integrate, sanitize, guardrail, persist. The `par` block is the delegation-supervisor-workers core — all three data workers run concurrently and the workflow joins their results. The sanitizerStep runs before guardrailStep; either failure routes to `BLOCKED`.

## State machine

`StockReport` moves `RESOLVING → IN_PROGRESS`, then to one of three terminals: `PUBLISHED` (sanitizer and guardrail both passed), `PARTIAL` (one or more workers timed out), or `BLOCKED` (sanitizer or guardrail failed). `PUBLISHED` accepts one further `EvalScored` self-transition when `EvalSampler` records a numeric accuracy score.

## Entity model

`RequestQueue` seeds one `StockReport` per query. A report owns at most one `TickerResolution`, one `PriceSummary`, one `NewsSummary`, one `FundamentalsSnapshot`, and one `IntegratedReport`. The view row mirrors the report with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
