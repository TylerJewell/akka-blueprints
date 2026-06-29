# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A research query enters through `DigestEndpoint` (`POST /api/digests`) or is dripped by `QuerySimulator` every 60 seconds. Either path writes a `QuerySubmitted` event onto `QueryQueue`. `QueryRequestConsumer` subscribes to those events and starts one `ResearchWorkflow` per submission, keyed by `digestId`.

The workflow is the supervisor. It asks `LiteratureCoordinator` to decompose the query into a scouting directive and interpretation question, fans the work out to `PaperScout` and `DomainAnalyst` in parallel, then asks `LiteratureCoordinator` to synthesise. Every transition is written as a command to `ResearchDigestEntity`, whose events project into `DigestView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a digest to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out, join, synthesise, citation guardrail, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The guardrail decides between the `SYNTHESISED` and `BLOCKED` terminals.

## State machine

`ResearchDigest` moves `QUEUED → SCANNING`, then to one of three terminals: `SYNTHESISED` (guardrail passed), `DEGRADED` (a worker timed out), or `BLOCKED` (citation guardrail failed). `SYNTHESISED` accepts one further `DigestEvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`QueryQueue` seeds one `ResearchDigest` per query. A digest owns at most one `PublicationBundle`, one `TrendReport`, and one `SynthesisedDigest`. The view row mirrors the digest with `Optional<T>` on every field that is null before its transition (Lesson 6).
