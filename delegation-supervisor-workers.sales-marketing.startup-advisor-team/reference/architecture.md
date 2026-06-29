# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A startup profile enters through `AdvisoryEndpoint` (`POST /api/advisory`) or is dripped by `ProfileSimulator` every 60 seconds. Either path writes a `ProfileSubmitted` event onto `ProfileQueue`. `ProfileConsumer` subscribes to those events and starts one `AdvisoryWorkflow` per submission, keyed by `sessionId`.

The workflow is the supervisor. It asks `AdvisorSupervisor` to decompose the profile into four work items, fans the work out to `MarketResearcher`, `GtmStrategist`, `ContentPlanner`, and `RoadmapAdvisor` in parallel, then asks `AdvisorSupervisor` to synthesise. Every state transition is written as a command to `AdvisorySessionEntity`, whose events project into `AdvisoryView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a session to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, four-way parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — all four specialists run concurrently and the workflow joins their results before calling synthesis. The guardrail decides between the `SYNTHESISED` and `BLOCKED` terminals.

## State machine

`AdvisorySession` moves `PLANNING → IN_PROGRESS`, then to one of three terminals: `SYNTHESISED` (guardrail passed), `DEGRADED` (at least one specialist timed out), or `BLOCKED` (guardrail failed). `SYNTHESISED` accepts one further `EvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`ProfileQueue` seeds one `AdvisorySession` per startup profile submission. A session owns at most one `MarketLandscape`, one `GtmStrategy`, one `ContentPlan`, one `ProductRoadmap`, and one `AdvisoryReport`. The view row mirrors the session with `Optional<T>` on every field that is null before its transition (Lesson 6).
