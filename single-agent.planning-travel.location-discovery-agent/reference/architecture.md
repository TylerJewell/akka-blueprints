# Architecture — location-discovery-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SearchEndpoint` accepts a submission and writes a `SearchSubmitted` event onto `SearchEntity`. The `CoordinateSanitizer` Consumer subscribes, redacts the raw GPS coordinates to a coarse bounding-box token, loads the matching candidate places from the in-process dataset, and writes the sanitized coordinate and place list back via `attachSanitized`. The same Consumer then starts a `DiscoveryWorkflow` instance. The workflow's `discoverStep` calls `LocationDiscoveryAgent` — the single AutonomousAgent — with the query and category filter as `TaskDef.instructions(...)` and the candidate place JSON as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`RecommendationGuardrail`) validates each candidate response. Once a recommendation passes, the workflow writes `RecommendationRecorded` and runs `EvaluationScorer` in `evalStep`. The score lands as `EvaluationScored`. `SearchView` projects every entity event into a read-model row; `SearchEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`EvaluationScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system pauses:

1. The `CoordinateSanitizer` subscription lag between `SearchSubmitted` and `CoordinateSanitized` — sub-second in normal operation, as the place candidate lookup is an in-process dataset scan.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `SearchEntity` every 1 s up to its 15 s timeout, advancing as soon as `search.sanitized().isPresent()` returns true.

The agent call itself is bounded by `discoverStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → COORDINATE_SANITIZED → DISCOVERING → RECOMMENDATION_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `DISCOVERING`. A `FAILED` search's prior data is preserved on the entity — the UI shows the partial state for the user.
- There is no `BOOKED` or `NAVIGATED` state. The recommendation is advisory; the user reads it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`SearchEntity` is the source of truth. It emits six event types. `SearchView` projects every event into a row used by the UI. `CoordinateSanitizer` subscribes to entity events to compute the sanitized coordinate. `DiscoveryWorkflow` both reads (`getSearch`) and writes (`markDiscovering`, `recordRecommendation`, `recordEvaluation`, `fail`) on the entity. The relationship between `LocationDiscoveryAgent` and `PlaceRecommendation` is "returns" — the agent's task result is the recommendation record.

## Defence-in-depth governance flow

For any recommendation that lands in the entity log, the search passed through:

1. **Coordinate sanitizer** — the model never sees the user's exact GPS position; the audit log retains the raw coordinates.
2. **LocationDiscoveryAgent** — one model call, one structured output against the candidate place list.
3. **before-agent-response guardrail** — hallucinated place ids, out-of-range scores, and empty results are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed recommendation still gets a 1–5 quality score so the user knows which results to trust at a glance.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
