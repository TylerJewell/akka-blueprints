# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system embeds the same mermaid sources on the Architecture tab.

## Component graph

The supervisor is `TripPlanningWorkflow`. Two paths start it: a direct `POST /api/trips` through `TripEndpoint`, and the background path where `TripRequestSimulator` drips canned requests into `InboundRequestQueue`, whose events `TripRequestConsumer` turns into workflow starts. The supervisor delegates in sequence to the three worker agents, writing each result to `TripEntity`. `TripsView` projects the entity events into the read model the endpoint queries and streams. `LocalExpertAgent` reaches the in-process `TravelSourceEndpoint` through its search and scrape tools; the scrape tool is fenced by the before-tool-call guard.

## Interaction sequence

The primary journey runs gather → draft → plan → finalize. The local-expert's scrape call passes through the before-tool-call guard (G1), which blocks non-allowlisted URLs before the call is made. The itinerary agent's response passes through the before-agent-response guard (G2), which rejects an itinerary that names places absent from the gathered insights, forcing a bounded retry. The supervisor assembles the final plan and completes the trip.

## State machine

`TripEntity` moves through `GATHERING_INSIGHTS → PLANNING_ITINERARY → PLANNING_LOGISTICS → COMPLETED`, with `FAILED` reachable from any working state when a step exhausts its retries. The terminal states are `COMPLETED` and `FAILED`.

## Entity model

`TripEntity` emits `TripRequested`, `InsightsGathered`, `ItineraryDrafted`, `LogisticsPlanned`, `TripCompleted`, and `TripFailed`, each folded into the `TripPlan` state. `TripsView` keeps one row per trip. `InboundRequestQueue` emits `TripRequestQueued`, consumed to start workflows.
