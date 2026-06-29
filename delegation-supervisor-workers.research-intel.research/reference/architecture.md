# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A topic enters through `ResearchEndpoint` (`POST /api/research`) or is dripped by `RequestSimulator` every 60 seconds. Either path writes a `TopicSubmitted` event onto `RequestQueue`. `ResearchRequestConsumer` subscribes to those events and starts one `ResearchWorkflow` per submission, keyed by `briefId`.

The workflow is the supervisor. It asks `ResearchCoordinator` to decompose the topic, fans the work out to `Researcher` and `Analyst` in parallel, then asks `ResearchCoordinator` to synthesise. Every transition is written as a command to `ResearchBriefEntity`, whose events project into `ResearchView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a brief to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The guardrail decides between the `SYNTHESISED` and `BLOCKED` terminals.

## State machine

`ResearchBrief` moves `PLANNING → IN_PROGRESS`, then to one of three terminals: `SYNTHESISED` (guardrail passed), `DEGRADED` (a worker timed out), or `BLOCKED` (guardrail failed). `SYNTHESISED` accepts one further `EvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`RequestQueue` seeds one `ResearchBrief` per topic. A brief owns at most one `FindingsBundle`, one `AnalyticalReport`, and one `SynthesisedBrief`. The view row mirrors the brief with `Optional<T>` on every field that is null before its transition (Lesson 6).
