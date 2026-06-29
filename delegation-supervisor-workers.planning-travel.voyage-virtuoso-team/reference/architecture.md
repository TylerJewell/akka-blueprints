# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A travel request enters through `TravelEndpoint` (`POST /api/travel`) or is dripped by `RequestSimulator` every 60 seconds. Either path writes a `TravelSubmitted` event onto `RequestQueue`. `TravelRequestConsumer` subscribes to those events and starts one `ItineraryWorkflow` per submission, keyed by `requestId`.

The workflow is the supervisor. It asks `ItineraryDirector` to decompose the request into four pillar briefs, then fans the work out to `FlightSpecialist`, `AccommodationSpecialist`, `ExperienceSpecialist`, and `LogisticsSpecialist` — all running concurrently. When all four (or the surviving subset after any timeout) return, the workflow asks `ItineraryDirector` to assemble the final itinerary. Every transition is written as a command to `TravelRequestEntity`, whose events project into `TravelRequestView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a request to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, four-way parallel fan-out, join, assemble, guardrail, persist. The `par` block is the delegation-supervisor-workers core — all four specialists run concurrently and the workflow joins their results. The guardrail decides between the `ASSEMBLED` and `BLOCKED` terminals.

## State machine

`TravelRequest` moves `PLANNING → IN_PROGRESS`, then to one of three terminals: `ASSEMBLED` (guardrail passed), `PARTIAL` (at least one specialist timed out), or `BLOCKED` (guardrail failed). `ASSEMBLED` accepts one further `EvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`RequestQueue` seeds one `TravelRequest` per submission. A request owns at most one `FlightPlan`, one `LodgingPlan`, one `ExperiencePlan`, one `LogisticsPlan`, and one `ItineraryPlan`. The view row mirrors the request with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
