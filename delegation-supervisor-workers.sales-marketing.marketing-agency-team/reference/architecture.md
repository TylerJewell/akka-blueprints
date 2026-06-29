# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A campaign brief enters through `CampaignEndpoint` (`POST /api/campaigns`) or is dripped by `RequestSimulator` every 60 seconds. Either path writes a `CampaignBriefSubmitted` event onto `RequestQueue`. `CampaignRequestConsumer` subscribes to those events and starts one `CampaignWorkflow` per submission, keyed by `planId`.

The workflow is the supervisor. It asks `CampaignDirector` to decompose the brief into a launch scope and a strategy brief, fans the work out to `WebsiteLauncher` and `StrategyAdvisor` in parallel, then asks `CampaignDirector` to synthesise. Every transition is written as a command to `CampaignPlanEntity`, whose events project into `CampaignView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a plan to score.

## Interaction sequence

The sequence diagram traces the primary journey: scope, parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — both specialists run concurrently and the workflow joins their results. The brand-safety guardrail decides between the `SYNTHESISED` and `BLOCKED` terminals.

## State machine

`CampaignPlan` moves `PLANNING → IN_PROGRESS`, then to one of three terminals: `SYNTHESISED` (guardrail passed), `DEGRADED` (a specialist timed out), or `BLOCKED` (guardrail failed). `SYNTHESISED` accepts one further `PlanEvalScored` self-transition when `EvalSampler` records a brand-quality score.

## Entity model

`RequestQueue` seeds one `CampaignPlan` per brief. A plan owns at most one `LaunchBrief`, one `StrategyFramework`, and one `SynthesisedPlan`. The view row mirrors the plan with `Optional<T>` on every field that is null before its transition (Lesson 6).
