# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A ticker enters through `AnalysisEndpoint` (`POST /api/analysis`) or is dripped by `TickerSimulator` every 90 seconds. Either path writes a `TickerSubmitted` event onto `TickerQueue`. `TickerRequestConsumer` subscribes to those events and starts one `AnalysisWorkflow` per submission, keyed by `reportId`.

The workflow is the supervisor. It asks `AnalysisCoordinator` to decompose the ticker into three subtasks, fans the work out to `FinancialsAgent`, `NewsAgent`, and `RatioAgent` in parallel, then asks `AnalysisCoordinator` to synthesise. Every transition is written as a command to `StockReportEntity`, whose events project into `StockReportView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a report to score.

The four agents are the only LLM-calling components. The workflow, entity, view, consumer, and timed actions are deterministic Akka primitives — no model calls outside the agents.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, three-way parallel fan-out, join, synthesise, guardrail, sanitizer, persist. The `par` block is the delegation-supervisor-workers core — all three workers run concurrently and the workflow joins their results. The guardrail and sanitizer decide between the `COMPLETE` and `BLOCKED` terminals.

The degrade path (not shown in the happy-path sequence) activates when any of the three parallel steps exceeds its 60-second timeout. The workflow transitions to `degradeStep`, which calls the Coordinator with whichever partial results arrived, and ends with `ReportDegraded`.

## State machine

`StockReport` moves `QUEUED → ANALYSING`, then to one of three terminals: `COMPLETE` (guardrail and sanitizer passed), `DEGRADED` (a worker timed out), or `BLOCKED` (guardrail or sanitizer failed). `COMPLETE` accepts one further `ReportEvalScored` self-transition when `EvalSampler` records a factuality score.

## Entity model

`TickerQueue` seeds one `StockReport` per submission. A report owns at most one `FinancialMetrics`, one `NewsSummary`, one `RatioSet`, and one `StockRecommendation`. The view row mirrors the report with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6), so the SSE stream never carries partially-initialized data under a non-null field name.
