# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A trip request enters through `TripEndpoint` (`POST /api/trips`) or is dripped by `TripRequestSimulator` every 90 seconds. Either path writes a `TripRequested` event onto `TripRequestQueue`. `TripRequestConsumer` subscribes to those events and starts one `TripWorkflow` per submission, keyed by `tripId`.

The workflow is the supervisor. It asks `TripCoordinator` to decompose the request into two parallel work items: a destination query for `DestinationScout` and a fare query for `FareAgent`. Both agents run concurrently; the workflow joins their outputs and asks `TripCoordinator` to assemble the final itinerary. Every lifecycle transition is written as a command to `TripEntity`, whose events project into `TripView`. The endpoint reads and streams the view for the App UI's live list. After assembly, the workflow pauses at a human-in-the-loop step and sends an `ApprovalRequested` signal to `TripEndpoint`, which surfaces confirm and decline buttons in the UI.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out, join, assemble, HITL pause, user confirmation, guardrail, booking. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The HITL step suspends the workflow durably. The before-tool-call guardrail runs after confirmation and before the booking tool fires, deciding between `CONFIRMED` and `BLOCKED`.

## State machine

`Trip` moves `PLANNING → RESEARCHING` when research starts, then to `AWAITING_APPROVAL` once the itinerary is assembled. From there, three outcomes are possible: `CONFIRMED` (user approved and guardrail passed), `DECLINED` (user rejected), or `BLOCKED` (guardrail failed). `DEGRADED` is reachable from `RESEARCHING` when a worker times out before the join. All terminal states (`CONFIRMED`, `DECLINED`, `BLOCKED`, `DEGRADED`) do not self-loop.

## Entity model

`TripRequestQueue` seeds one `Trip` per submission. A trip accumulates at most one `DestinationBrief`, one `FareEstimate`, and one `TripItinerary` across its lifecycle. Each of these optional payloads is absent until its corresponding event fires; the view row mirrors the entity with `Optional<T>` on every nullable field (Lesson 6) so Jackson serialises a clean null on the wire before the transition.
