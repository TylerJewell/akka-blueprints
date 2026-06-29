# Architecture — surprise-trip-planner

Narrative around the four diagrams in `PLAN.md`. The generated UI renders the same diagrams on the Architecture tab.

## Component graph

A request enters either from the App UI (`POST /api/trips`) or from `RequestSimulator`, which drips canned preference sets. Both write to `InboundRequestQueue`. `TripRequestConsumer` subscribes to that entity's events and starts one `TripWorkflow` per request. The workflow delegates to the three worker agents, then to `TripSupervisorAgent` for assembly, recording each result on `TripEntity`. `TripsView` projects entity events into a read model the `TripEndpoint` queries and streams. `StalledTripMonitor` escalates trips that sit unfinished.

The delegation-supervisor-workers form lives in the workflow: the supervisor never does worker work; it decomposes and assembles. Solid arrows are synchronous commands, dashed arrows are event subscriptions, dotted arrows are scheduled ticks.

## Interaction sequence

The primary journey runs request → destination → logistics → activities → assemble. Each worker step calls `runSingleTask(...)` and blocks on `forTask(taskId).result(...)`. The two guardrails fire around assembly: G1 (before-tool-call) inside each worker step, G2 (before-agent-response) before the supervisor's assembled itinerary is persisted. The destination is hidden until the trip reaches `READY`.

## State machine

`TripEntity` moves `REQUESTED → RESEARCHED → READY` on the happy path. From `REQUESTED` or `RESEARCHED` a stalled trip can move to `ESCALATED`, and a step that exhausts its retries moves the trip to `FAILED`. `READY`, `ESCALATED`, and `FAILED` are terminal.

## Entity model

`InboundRequestQueue` spawns trips. `TripEntity` emits eight event types and projects to `TripsView`. The view row is the `Trip` record itself, with every nullable lifecycle field declared `Optional<T>` so the view materializer accepts rows written before those events fire.
