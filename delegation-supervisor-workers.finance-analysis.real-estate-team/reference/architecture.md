# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A property deal enters through `DealEndpoint` (`POST /api/deals`) or is dripped by `DealSimulator` every 60 seconds. Either path writes a `DealSubmitted` event onto `DealQueue`. `DealRequestConsumer` subscribes to those events and starts one `EvaluationWorkflow` per submission, keyed by `dealId`.

The workflow is the supervisor. Its first step is a pure Java sanitizer: it checks `DealSubmission.propertyType` against a hardcoded exclusion list and routes excluded deals to `DealRejected` without touching any AutonomousAgent. For passing submissions, the workflow asks `DealCoordinator` to scope the deal into an `EvaluationPlan`, fans the work out to `MarketSpecialist` and `FinanceSpecialist` in parallel, then asks `DealCoordinator` to synthesise an `InvestmentRecommendation`. Every transition is written as a command to `DealEntity`, whose events project into `DealView`. The endpoint reads and streams the view; `RecommendationEvalSampler` reads it to pick a deal to score.

## Interaction sequence

The sequence diagram traces the primary journey: sanitizer check, scope, parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — both specialists run concurrently and the workflow joins their results. The guardrail decides between the `RECOMMENDED` and `BLOCKED` terminals. The early-exit branch shows the sanitizer routing an excluded property type directly to `REJECTED`.

## State machine

`DealRecord` moves `SCOPING → REJECTED` if the sanitizer fires (terminal). Otherwise it moves `SCOPING → EVALUATING`, then to one of three terminals: `RECOMMENDED` (guardrail passed), `DEGRADED` (a specialist timed out), or `BLOCKED` (guardrail failed). `RECOMMENDED` accepts one further `RecommendationScored` self-transition when `RecommendationEvalSampler` records a quality score.

## Entity model

`DealQueue` seeds one `DealRecord` per submission. A deal owns at most one `MarketReport`, one `FinancialModel`, and one `InvestmentRecommendation`. The `DealView` row mirrors the deal with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). `MARKET_REPORT` and `FINANCIAL_MODEL` are attached independently as each specialist returns; the join step in the workflow waits for both before calling `DealCoordinator.SYNTHESISE`.
