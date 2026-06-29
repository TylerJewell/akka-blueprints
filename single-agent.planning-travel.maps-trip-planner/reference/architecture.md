# Architecture — maps-trip-planner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `PlanningEndpoint` accepts a trip submission and writes a `TripRequestSubmitted` event onto `TripRequestEntity`. The `RequestScreener` Consumer subscribes, checks the requested destinations against the blocked list, and writes either a screen pass (via `attachScreen`) or a block (via `block`) back to the entity. On a pass, the same Consumer starts a `PlanningWorkflow` instance.

The workflow's `planStep` calls `TripPlannerAgent` — the single AutonomousAgent — with the trip request details as `TaskDef.instructions(...)`. The agent calls `MockMapsClient` tool stubs (geocode, directions, place-details) to resolve places and transit times, then returns a `TripItinerary`. The agent's `before-agent-response` guardrail (`ItineraryGuardrail`) validates each candidate response. Once an itinerary passes, the workflow writes `ItineraryRecorded` and runs `CoverageScorer` in `evalStep`. The score lands as `EvaluationScored`. `ItineraryView` projects every entity event into a read-model row; `PlanningEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The on-decision evaluator (`CoverageScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system waits:

1. The `RequestScreener` subscription lag between `TripRequestSubmitted` and `RequestScreened` — sub-second in normal operation.
2. The `awaitScreenedStep` polling loop inside the workflow — polls `TripRequestEntity` every 1 s up to its 15 s timeout, advancing as soon as `plan.screen().isPresent()` and `plan.status() != BLOCKED`.

The agent call itself is bounded by `planStep`'s 90 s timeout to accommodate LLM latency plus the Maps tool round-trips. The `evalStep` is synchronous and finishes in milliseconds.

## State machine

Seven states. The interesting paths:

- The happy path is `SUBMITTED → SCREENED → PLANNING → ITINERARY_RECORDED → EVALUATED`.
- The blocked path is `SUBMITTED → BLOCKED` (terminal). The screener detected a restricted destination. No agent call is made; the entity preserves the original request for audit.
- Two failure transitions land in `FAILED`: a screener error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `PLANNING`.
- There is no `BOOKED` or `CONFIRMED` state. The itinerary is advisory; the traveller reads it and books separately. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`TripRequestEntity` is the source of truth. It emits seven event types. `ItineraryView` projects every event into a row used by the UI. `RequestScreener` subscribes to entity events to compute the screen result and decide whether to start the workflow. `PlanningWorkflow` both reads (`getTripPlan`) and writes (`markPlanning`, `recordItinerary`, `recordEvaluation`, `fail`) on the entity. The relationship between `TripPlannerAgent` and `TripItinerary` is "returns" — the agent's task result is the itinerary record.

## Defence-in-depth governance flow

For any itinerary that lands in the entity log, the request passed through:

1. **Destination screener** — restricted destinations are blocked before any LLM call; the audit log retains the raw request.
2. **TripPlannerAgent** — one model call, one structured output, backed by Maps tool stubs for geocoding and routing.
3. **before-agent-response guardrail** — empty stop lists, non-positive transit durations, and day-count mismatches are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed itinerary still gets a 1–5 coverage score so the reviewer knows which outputs to trust at a glance.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
