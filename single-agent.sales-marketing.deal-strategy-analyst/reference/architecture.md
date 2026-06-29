# Architecture — deal-strategy-analyst

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `DealEndpoint` accepts a deal submission and writes a `DealSubmitted` event onto `DealEntity`. The `DealContextSanitizer` Consumer subscribes, redacts PII and confidential identifiers, and writes the sanitized context back via `attachSanitized`. The same Consumer then starts an `AnalysisWorkflow` instance. The workflow's `analyseStep` calls `DealStrategyAgent` — the single AutonomousAgent — with a structured deal brief as `TaskDef.instructions(...)` and the sanitized context as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`RecommendationGuardrail`) validates each candidate response. Once a recommendation passes, the workflow writes `RecommendationRecorded` and runs `RecommendationScorer` in `evalStep`. The score lands as `EvaluationScored`. `DealView` projects every entity event into a read-model row; `DealEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`RecommendationScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct pauses:

1. The `DealContextSanitizer` subscription lag between `DealSubmitted` and `ContextSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `DealEntity` every 1 s up to its 15 s timeout, advancing as soon as `deal.sanitized().isPresent()` returns true.

The agent call itself is bounded by `analyseStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → ANALYSING → RECOMMENDATION_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `ANALYSING`. A `FAILED` deal's prior data is preserved on the entity — the UI shows the partial state for the rep.
- There is no `ACCEPTED` or `REJECTED` state. The recommendation is advisory; the sales rep reads it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`DealEntity` is the source of truth. It emits six event types. `DealView` projects every event into a row used by the UI. `DealContextSanitizer` subscribes to entity events to compute the sanitized form. `AnalysisWorkflow` both reads (`getDeal`) and writes (`markAnalysing`, `recordRecommendation`, `recordEvaluation`, `fail`) on the entity. The relationship between `DealStrategyAgent` and `StrategyRecommendation` is "returns" — the agent's task result is the recommendation record.

## Defence-in-depth governance flow

For any recommendation that lands in the entity log, the deal context passed through:

1. **PII sanitizer** — the model never sees buyer email addresses, phone numbers, or NDA-restricted company names; the audit log retains the raw form.
2. **DealStrategyAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — bad parses, unknown stakeholder roles, out-of-enum action types, and duplicate priorities are caught before the response leaves the agent's loop.
4. **On-decision evaluator** — every well-formed recommendation still gets a 1–5 actionability score so the rep knows which plans to trust at a glance.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
