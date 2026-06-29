# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A startup brief enters through `AdvisoryEndpoint` (`POST /api/advisory`) or is dripped by `BriefSimulator` every 60 seconds. Either path writes a `BriefSubmitted` event onto `BriefQueue`. `BriefConsumer` subscribes to those events and starts one `AdvisoryWorkflow` per submission, keyed by `sessionId`.

The workflow is the supervisor. It asks `AdvisorCoordinator` to decompose the brief, fans the work out to `MarketResearcher` and `GrowthAnalyst` in parallel, then asks `AdvisorCoordinator` to synthesise. Both workers operate through an MCP-exposed tool surface; the before-tool-call guardrail intercepts every tool call and enforces the allow-list before execution proceeds. Every state transition is written as a command to `AdvisorySessionEntity`, whose events project into `AdvisoryView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a session to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out, join, synthesise, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The before-tool-call guardrail annotation on each worker branch shows where the allow-list check fires before any tool executes.

## State machine

`AdvisorySession` moves `SCOPING → IN_PROGRESS`, then to one of two terminals: `ADVISED` (guidance synthesised successfully) or `DEGRADED` (a worker timed out). `ADVISED` accepts one further `GuidanceEvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`BriefQueue` seeds one `AdvisorySession` per submitted brief. A session owns at most one `MarketSnapshot`, one `ChannelPlan`, and one `GtmGuidance`. The view row mirrors the session with `Optional<T>` on every field that is null before its transition (Lesson 6). The ER diagram surfaces the one-to-one ownership across the key aggregates.
