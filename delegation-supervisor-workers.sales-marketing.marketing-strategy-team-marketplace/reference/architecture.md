# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A campaign objective enters through `CampaignEndpoint` (`POST /api/campaigns`) or is dripped by `CampaignSimulator` every 60 seconds. Either path writes an `ObjectiveSubmitted` event onto `ObjectiveQueue`. `CampaignRequestConsumer` subscribes to those events and starts one `CampaignWorkflow` per submission, keyed by `briefId`.

The workflow is the supervisor. It asks `CampaignDirector` to decompose the objective into a `StrategyPlan`, then fans the work out to `MarketResearcher`, `AudienceTargeter`, `MessageStrategist`, and `ChannelPlanner` in parallel. When all four return (or any times out), it asks `CampaignDirector` to assemble a unified brief. Every transition is written as a command to `CampaignBriefEntity`, whose events project into `CampaignView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a brief to score.

## Interaction sequence

The sequence diagram traces the primary journey: plan, parallel fan-out to four workers, join, assemble, guardrail, persist. The `par` block is the delegation-supervisor-workers core — all four workers run concurrently and the workflow joins their results before passing them to the director for assembly. The compliance guardrail decides between the `ASSEMBLED` and `BLOCKED` terminals.

## State machine

`CampaignBrief` moves `PLANNING → IN_PROGRESS`, then to one of three terminals: `ASSEMBLED` (guardrail passed), `DEGRADED` (at least one worker timed out and assembly proceeded from partial input), or `BLOCKED` (guardrail found a brand or legal violation). `ASSEMBLED` accepts one further `CampaignEvalScored` self-transition when `EvalSampler` records a quality score.

## Entity model

`ObjectiveQueue` seeds one `CampaignBrief` per objective. A brief owns at most one `MarketInsightsBundle`, one `AudienceProfile`, one `MessagingGuide`, one `ChannelPlan`, and one `AssembledBrief`. The view row mirrors the brief with `Optional<T>` on every field that is null before its transition (Lesson 6).
