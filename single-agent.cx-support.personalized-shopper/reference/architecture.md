# Architecture — personalized-shopper

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ShoppingEndpoint` accepts a session submission and writes a `SessionStarted` event onto `ShoppingSessionEntity`. The `ProfileSanitizer` Consumer subscribes, strips PII from the preference profile, and writes the sanitized profile back via `attachSanitized`. The same Consumer then starts a `RecommendationWorkflow` instance. The workflow's `recommendStep` calls `ShoppingAdvisorAgent` — the single AutonomousAgent — with the product catalog as `TaskDef.instructions(...)` and the sanitized preference profile as a `TaskDef.attachment(...)`. Once a recommendation set is returned, the workflow writes `RecommendationsRecorded` and runs `FreshnessScorer` in `freshnessStep`. The freshness score lands as `FreshnessScored`. `RecommendationView` projects every entity event into a read-model row; `ShoppingEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The freshness scorer (`FreshnessScorer`) is a deterministic rule-based function — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two pauses are worth noting:

1. The `ProfileSanitizer` subscription lag between `SessionStarted` and `ProfileSanitized` — sub-second in normal operation, bounded by the Consumer's event delivery.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `ShoppingSessionEntity` every 1 s up to its 15 s timeout, advancing as soon as `session.sanitized().isPresent()` returns true.

The agent call is bounded by `recommendStep`'s 60 s timeout. The `freshnessStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The notable paths:

- The happy path is `STARTED → PROFILE_SANITIZED → RECOMMENDING → RECOMMENDATIONS_READY → FRESHNESS_SCORED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `STARTED`, and an agent error (or iteration-budget exhaustion) during `RECOMMENDING`. A `FAILED` session retains all data collected up to the point of failure — the UI shows the partial state so a support user can diagnose what happened.
- There is no `PURCHASED` or `CART_ADDED` state. The recommendation is advisory; the shopper decides what to add to their cart outside this system. The blueprint deliberately stops at `FRESHNESS_SCORED`.

## Entity model

`ShoppingSessionEntity` is the source of truth. It emits six event types. `RecommendationView` projects every event into a row used by the UI. `ProfileSanitizer` subscribes to entity events to compute the sanitized profile. `RecommendationWorkflow` both reads (`getSession`) and writes (`markRecommending`, `recordRecommendations`, `recordFreshness`, `fail`) on the entity. The relationship between `ShoppingAdvisorAgent` and `RecommendationSet` is "returns" — the agent's task result is the recommendation-set record.

## Governance flow

For any recommendation set that lands in the entity log, the preference profile passed through:

1. **PII sanitizer** — the model never sees the shopper's name, email, phone, or loyalty-card number; the audit log retains the raw form on the entity.
2. **ShoppingAdvisorAgent** — one model call, one structured output, against a catalog and a preference signal set that contains no personal identifiers.
3. **FreshnessScorer** — every recommendation set gets a 1–5 freshness score so the shopper (and any downstream system) can see how much of the recommendation is in-stock and current-season at the moment it was produced.

Each layer is independent. Removing the sanitizer opens a clear PII-to-LLM gap; removing the freshness scorer removes the only post-hoc signal about recommendation quality relative to current catalog state.
