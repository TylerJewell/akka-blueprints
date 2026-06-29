# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A holdings submission enters through `PortfolioEndpoint` (`POST /api/portfolio`) or is dripped by `HoldingsSimulator` every 60 seconds. Both paths write a `HoldingsSubmitted` event onto `HoldingsQueue`. Before the event is written, however, `PortfolioWorkflow.sanitizeStep` normalizes the submitted sector tag against `SectorRegistry`. A prohibited sector terminates the workflow immediately — no event is written, no agent is invoked.

For permitted sectors, `HoldingsConsumer` subscribes to `HoldingsQueue` events and starts one `PortfolioWorkflow` per submission, keyed by `reportId`.

The workflow is the supervisor. It asks `PortfolioCoordinator` to decompose the holdings into an `AnalysisPlan`, fans the work out to `HoldingsAnalyst` and `MarketContextAgent` in parallel, then asks `PortfolioCoordinator` to consolidate. Every lifecycle transition is written as a command to `PortfolioReportEntity`, whose events project into `PortfolioView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a report to score.

## Interaction sequence

The sequence diagram traces the primary journey: sanitize, plan, parallel fan-out, join, consolidate, guardrail, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. An early-exit branch shows the prohibited-sector rejection path. The guardrail decides between the `CONSOLIDATED` and `BLOCKED` terminals.

## State machine

`PortfolioReport` starts in `PLANNING`. The PLANNING state can be short-circuited to `BLOCKED` immediately if the sector sanitizer fires. For valid sectors, the report moves to `IN_PROGRESS` once the `AnalysisPlan` is ready, then to one of three terminals: `CONSOLIDATED` (guardrail passed), `DEGRADED` (a worker timed out), or `BLOCKED` (guardrail failed). `CONSOLIDATED` accepts one further `ReportEvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`HoldingsQueue` seeds one `PortfolioReport` per submission. A report owns at most one `PositionAssessment`, one `MarketContext`, and one `ConsolidatedReport`. The view row mirrors the report with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). The sector field is always present because it is normalized before the report is created.
