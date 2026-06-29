# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A contract enters through `ReviewEndpoint` (`POST /api/reviews`) or is dripped by `ContractSimulator` every 90 seconds. Either path writes a `ContractSubmitted` event onto `ContractQueue`. `ContractQueueConsumer` subscribes to those events and starts one `ReviewWorkflow` per submission, keyed by `reviewId`.

The workflow is the supervisor. Its first act is a deterministic legal-sector sanitizer step: contracts with prohibited content are rejected immediately, with no agent ever called. For clean contracts, it asks `ReviewSupervisor` to decompose the work, then fans out to `ClauseAnalyst`, `RiskScorer`, and `Redliner` in parallel. After joining, it asks `ReviewSupervisor` to consolidate. A before-agent-response guardrail vets the package; on pass, the workflow suspends in an approval step waiting for a lawyer. Every state transition is written as a command to `ContractReviewEntity`, whose events project into `ReviewView`. The endpoint reads and streams the view; `ReviewSampler` reads it to pick approved reviews to score.

## Interaction sequence

The sequence diagram traces the primary journey: sanitize, plan, parallel fan-out, join, consolidate, guardrail, suspend for lawyer approval, finalise. The `par` block is the delegation-supervisor-workers core — all three workers run concurrently and the workflow joins their results. The guardrail decides between `AWAITING_APPROVAL` and `BLOCKED`. The approval step distinguishes between `APPROVED` and `REJECTED` based on the lawyer's explicit action.

## State machine

`ContractReview` moves `QUEUED → IN_REVIEW`, then to one of two intermediate states: `AWAITING_APPROVAL` (guardrail passed, lawyer needed) or `BLOCKED` (guardrail failed). From `AWAITING_APPROVAL`, a lawyer action moves the review to `APPROVED` or `REJECTED`. A review also reaches `DEGRADED` if a worker times out or the approval window lapses. `APPROVED` accepts one further `ReviewEvalScored` self-transition when `ReviewSampler` records a quality score. `REJECTED` is also reachable directly from `QUEUED` when the sanitizer fires.

## Entity model

`ContractQueue` seeds one `ContractReview` per submitted contract. A review owns at most one `ClauseSummary`, one `RiskReport`, one `RedlineSet`, and one `ReviewPackage`. The view row mirrors the review with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). `ClauseSummary`, `RiskReport`, and `RedlineSet` are independent outputs that arrive from three parallel workers — they are attached to the entity individually as they complete, then joined for consolidation.
