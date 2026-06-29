# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A market question enters through `AnalysisEndpoint` (`POST /api/analysis`) or is dripped by `RequestSimulator` every 60 seconds. Either path writes a `QuestionSubmitted` event onto `RequestQueue`. `MarketRequestConsumer` subscribes to those events and starts one `AnalysisWorkflow` per submission, keyed by `reportId`.

The workflow is the supervisor. It asks `EconomicsCoordinator` to frame the question into a `ResearchPlan`, fans the work out to `DataCollector` and `Economist` in parallel, then asks `EconomicsCoordinator` to synthesise. Every transition is written as a command to `AnalysisReportEntity`, whose events project into `AnalysisView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a report to score.

## Interaction sequence

The sequence diagram traces the primary journey: frame, parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The guardrail decides between the `PUBLISHED` and `BLOCKED` terminals.

## State machine

`AnalysisReport` moves `FRAMING → IN_PROGRESS`, then to one of three terminals: `PUBLISHED` (guardrail passed), `DEGRADED` (a worker timed out), or `BLOCKED` (guardrail failed). `PUBLISHED` accepts one further `ReportEvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`RequestQueue` seeds one `AnalysisReport` per question. A report owns at most one `IndicatorBundle`, one `EconomicInterpretation`, and one `SynthesisedReport`. The view row mirrors the report with `Optional<T>` on every field that is null before its transition (Lesson 6).
