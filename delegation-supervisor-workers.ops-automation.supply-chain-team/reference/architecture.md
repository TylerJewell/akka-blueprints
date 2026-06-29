# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A demand signal enters through `SupplyEndpoint` (`POST /api/supply`) or is dripped by `DemandSimulator` every 60 seconds. Either path writes a `DemandSignalSubmitted` event onto `OrderQueueEntity`. `DemandSignalConsumer` subscribes to those events and starts one `OptimizationWorkflow` per submission, keyed by `orderId`.

The workflow is the supervisor. It asks `DemandCoordinator` to decompose the demand signal, fans the work out to `StockAnalyst` and `LogisticsPlanner` in parallel, then asks `DemandCoordinator` to synthesise. Every transition is written as a command to `SupplyOrderEntity`, whose events project into `SupplyOrderView`. The endpoint reads and streams the view; `FulfillmentEvalSampler` reads it to pick an order to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The guardrail decides between the `RECOMMENDED` and `BLOCKED` terminals.

## State machine

`SupplyOrder` moves `PENDING → ANALYZING`, then to one of three terminals: `RECOMMENDED` (guardrail passed), `DEGRADED` (a worker timed out), or `BLOCKED` (guardrail failed). `RECOMMENDED` accepts one further `RecommendationScored` self-transition when `FulfillmentEvalSampler` records a score.

## Entity model

`OrderQueueEntity` seeds one `SupplyOrder` per demand signal. An order owns at most one `StockAssessment`, one `RoutePlan`, and one `SupplyRecommendation`. The view row mirrors the order with `Optional<T>` on every field that is null before its transition (Lesson 6).
