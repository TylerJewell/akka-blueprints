# Architecture — gemma-food-tour-guide

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on one decision-making LLM call. `TourEndpoint` accepts a tour request and writes a `TourRequested` event onto `TourRequestEntity`. The `PreferenceValidator` Consumer subscribes, normalizes the free-text notes field to canonical dietary category labels, and writes the validated preferences back via `attachValidated`. The same Consumer then starts a `TourWorkflow` instance. The workflow's `generateStep` calls `FoodTourAgent` — the single AutonomousAgent — with the validated preferences as `TaskDef.instructions(...)` and the seeded city profile as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ItineraryGuardrail`) validates each candidate response. Once an itinerary passes, the workflow writes `ItineraryRecorded` and runs `CoverageScorer` in `evalStep`. The score lands as `CoverageScored`. `TourView` projects every entity event into a read-model row; `TourEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `CoverageScorer` is a deterministic, rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system awaits something:

1. The `PreferenceValidator` subscription lag between `TourRequested` and `PreferencesValidated` — sub-second in normal operation.
2. The `awaitValidatedStep` polling loop inside the workflow — polls `TourRequestEntity` every 1 s up to its 15 s timeout, advancing as soon as `request.validated().isPresent()` returns true.

The agent call is bounded by `generateStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The notable paths:

- The happy path is `REQUESTED → PREFERENCES_VALIDATED → GENERATING → ITINERARY_RECORDED → SCORED`.
- Two failure transitions land in `FAILED`: a validator error during `REQUESTED`, and an agent error (or guardrail-exhaustion) during `GENERATING`. A `FAILED` tour request's prior data is preserved on the entity — the UI shows the partial state for the traveler.
- There is no `BOOKED` or `CONFIRMED` state. The itinerary is advisory; the traveler consults it and books independently. The blueprint deliberately stops at `SCORED`.

## Entity model

`TourRequestEntity` is the source of truth. It emits six event types. `TourView` projects every event into a row used by the UI. `PreferenceValidator` subscribes to entity events to compute the validated form. `TourWorkflow` both reads (`getTourRequest`) and writes (`markGenerating`, `recordItinerary`, `recordCoverage`, `fail`) on the entity. The relationship between `FoodTourAgent` and `TourItinerary` is "returns" — the agent's task result is the itinerary record.

## Governance flow

For any itinerary that lands in the entity log, the request passed through:

1. **Preference normalizer** — the model sees only canonical category labels; raw health notes are not forwarded.
2. **FoodTourAgent** — one model call, one structured itinerary.
3. **before-agent-response guardrail** — out-of-range day indices, unrecognized meal slot values, and uncovered dietary categories are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed itinerary still gets a 1–5 coverage score so the traveler and platform operator know which plans are thin before the trip begins.

Each step is independent. Removing one of them opens a specific gap the others do not silently cover.
