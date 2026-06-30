# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A trip request enters through `TripEndpoint` (`POST /api/trips`) or is dripped by `TripRequestSimulator` every 90 seconds. Either path writes a `TripRequestSubmitted` event onto `TripRequestQueue`. `TripRequestConsumer` subscribes to those events and starts one `TripWorkflow` per submission, keyed by `tripId`.

The workflow is the supervisor. It asks `TripSupervisor` to decompose the request, fans the work out to `DestinationResearcher`, `ItineraryPlanner`, and `BookingAgent` in parallel, then asks `TripSupervisor` to compile the three outputs into a `TripPlan`. Every lifecycle transition is written as a command to `TripEntity`, whose events project into `TripView`. The endpoint reads and streams the view; `EvalSampler` reads it to find confirmed trips to score. `ApprovalEntity` holds the HITL token that pairs the paused workflow to the user's approval signal.

## Interaction sequence

The sequence diagram traces the primary happy-path journey: decompose, parallel fan-out to three workers, join, compile, guardrail, issue approval token, workflow pauses, user approves, confirm. The `par` block is the delegation-supervisor-workers core — all three workers run concurrently and the workflow joins their results. The guardrail decides between the `AWAITING_APPROVAL` path and the `BLOCKED` terminal. The workflow then pauses until the endpoint delivers the user's decision.

## State machine

`Trip` moves `PLANNING → IN_PROGRESS`, then through the guardrail to `AWAITING_APPROVAL`. From there the user drives it to `CONFIRMED` or `REJECTED`. Two additional terminals exist: `DEGRADED` (a worker timed out; supervisor compiled from partial inputs) and `BLOCKED` (the budget guardrail fired). `CONFIRMED` accepts one further `PlanEvalScored` self-transition when `EvalSampler` records a quality score.

## Entity model

`TripRequestQueue` seeds one `Trip` per request. A trip owns at most one `DestinationNotes`, one `Itinerary`, one list of `BookingProposal` records, and one compiled `TripPlan`. `ApprovalEntity` holds the token that links the paused workflow to the user-approval HTTP call. The view row mirrors the trip with `Optional<T>` on every field that is null before its transition (Lesson 6).
