# Architecture — pokemon-advisor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `AdvisoryEndpoint` accepts a roster submission and writes a `RosterSubmitted` event onto `RosterEntity`. The `RosterValidator` Consumer subscribes, checks roster legality (no duplicate species, correct slot count for the format), and writes the validated roster back via `attachValidated`. If validation fails (e.g., duplicate species), the Consumer calls `fail` on the entity instead and does not start the workflow. On success, the same Consumer starts an `AdvisoryWorkflow` instance.

The workflow's `adviseStep` calls `TeamAdvisorAgent` — the single AutonomousAgent — with the battle format constraints as `TaskDef.instructions(...)` and the validated roster as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`RecommendationGuardrail`) validates each candidate response for structural correctness. Once a recommendation passes, the workflow writes `RecommendationRecorded` and runs `CoverageScorer` in `scoreStep`. The score lands as `CoverageScored`. `AdvisoryView` projects every entity event into a read-model row; `AdvisoryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`CoverageScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct waiting moments:

1. The `RosterValidator` subscription lag between `RosterSubmitted` and `RosterValidated` — sub-second in normal operation.
2. The `awaitValidatedStep` polling loop inside the workflow — polls `RosterEntity` every 1 s up to its 15 s timeout, advancing as soon as `advisory.validated().isPresent()` returns true.

The agent call itself is bounded by `adviseStep`'s 60 s timeout. The `scoreStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → VALIDATED → ADVISING → RECOMMENDATION_RECORDED → SCORED`.
- Two failure transitions land in `FAILED`: a validation error during `SUBMITTED` (duplicate species or out-of-range slot count), and an agent error (or guardrail-exhaustion) during `ADVISING`. A `FAILED` advisory's prior data is preserved on the entity — the UI shows the partial state and the failure reason.
- There is no `LOCKED` or `COMMITTED` state. The recommendation is advisory; the trainer reads it and acts in the game client. The blueprint stops at `SCORED`.

## Entity model

`RosterEntity` is the source of truth. It emits six event types. `AdvisoryView` projects every event into a row used by the UI. `RosterValidator` subscribes to entity events to run the legality check. `AdvisoryWorkflow` both reads (`getAdvisory`) and writes (`markAdvising`, `recordRecommendation`, `recordCoverageScore`, `fail`) on the entity. The relationship between `TeamAdvisorAgent` and `TeamRecommendation` is "returns" — the agent's task result is the recommendation record.

## Governance flow

For any recommendation that lands in the entity log, the roster passed through:

1. **RosterValidator** — legality checked before any LLM call; duplicate species and format-violating rosters are rejected at this stage.
2. **TeamAdvisorAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — bad parses, out-of-enum roles, missing slots, and severity violations are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed recommendation still gets a 1–5 coverage score so the trainer knows how thoroughly the team was analysed.

Each step is independent. Removing one of them opens an explicit gap the others do not cover.
