# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A candidate site enters through `LocationEndpoint` (`POST /api/locations`) or is dripped by `SiteSimulator` every 60 seconds. Either path writes a `SiteSubmitted` event onto `SiteQueue`. `SiteRequestConsumer` subscribes to those events and starts one `LocationWorkflow` per submission, keyed by `siteId`.

The workflow is the supervisor. It asks `LocationCoordinator` to decompose the candidate site into a `ScoringPlan`, fans the work out to `MarketAnalyst` and `DemographicsAnalyst` in parallel, then asks `LocationCoordinator` to synthesise a `SiteRecommendation`. Every transition is written as a command to `CandidateSiteEntity`, whose events project into `LocationView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a finished evaluation to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — both analysts run concurrently and the workflow joins their results before asking the coordinator to synthesise. The guardrail decides between the terminal states.

## State machine

`CandidateSite` moves `SCORING → IN_PROGRESS`, then to one of four terminals: `RECOMMENDED` (guardrail passed, score ≥ 0.60), `NOT_RECOMMENDED` (guardrail passed, score < 0.60), `DEGRADED` (an analyst timed out), or `BLOCKED` (guardrail failed). Both `RECOMMENDED` and `NOT_RECOMMENDED` accept one further `EvalScored` self-transition when `EvalSampler` records a quality score.

## Entity model

`SiteQueue` seeds one `CandidateSite` per submission. A site owns at most one `MarketAssessment`, one `DemographicAssessment`, and one `SiteRecommendation`. The view row mirrors the site with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
