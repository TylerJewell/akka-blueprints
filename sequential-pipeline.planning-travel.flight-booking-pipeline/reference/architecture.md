# Architecture — flight-booking-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence, with a workflow-level pause between SELECT and BOOK to collect user confirmation. `BookingEndpoint` accepts a `{origin, destination, travelDate}` POST, writes `BookingCreated` onto `BookingEntity`, and starts `BookingPipelineWorkflow` keyed by `"pipeline-" + bookingId`. The workflow's first step (`searchStep`) emits `SearchStarted`, then calls `BookingAgent` with `TaskDef.taskType(SEARCH_FLIGHTS)` and a `phase = SEARCH` metadata tag. The agent invokes `SearchTools.searchFlights` and `SearchTools.getFareDetails`; every call passes through `BookingGuardrail` first. Once the agent returns a `FlightOfferSet`, the workflow writes `FlightsFound` onto the entity and advances to `selectStep` — same pattern, the SELECT task carries `phase = SELECT`. After `SeatSelected` lands, the workflow emits `AwaitingConfirmation` and suspends at `confirmStep` until the user clicks **Confirm booking** in the UI. The user's `POST /api/bookings/{id}/confirm` call writes `BookingConfirmed` onto the entity, which unblocks the workflow. `bookStep` then runs the BOOK task with `phase = BOOK`. After `BookingCommitted` lands, `BookingView` projects the final row to the UI. `BookingEndpoint` serves the read model over REST and SSE.

The graph has exactly one LLM-calling component. The confirmation wait in `confirmStep` is a deterministic workflow pause — no LLM call, no second agent. That is what makes this a faithful **single-agent sequential-pipeline** example with an application-level HITL gate.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. **Task boundary = dependency contract.** Between `searchStep` and `selectStep`, the workflow writes `FlightsFound` onto the entity. `selectStep` then reads `flightOffers` from the entity to build the SELECT task's instruction context. The agent never sees search-phase context inside the select task's conversation; the typed handoff is the only path information travels.

2. **Confirmation gate is enforced at two layers.** The workflow pauses at `confirmStep` and does not advance to `bookStep` without `BookingConfirmed`. Even if the agent loop somehow attempted `commitReservation` before that event, `BookingGuardrail` would reject the call with `hitl-violation`. Both layers must fail independently before a booking could commit without user approval.

3. **Every tool call is filtered through `BookingGuardrail`.** The guardrail reads the in-flight task's `phase` metadata and the current `BookingEntity.status`, and applies the per-phase accept matrix plus the confirmation precondition. A misordered call is rejected before the tool body executes.

The agent calls are bounded by per-step timeouts (60 s on search / select / book). `confirmStep` has a 1-hour timeout — a human confirmation window.

## State machine

Ten states. The key paths:

- The happy path walks `CREATED → SEARCHING → SEARCH_DONE → SELECTING → SEAT_SELECTED → AWAITING_CONFIRMATION → CONFIRMED → BOOKING → COMMITTED`.
- `AWAITING_CONFIRMATION` is the only non-terminal pause state. The workflow stays here until the user confirms. If the 1-hour timeout elapses without confirmation, the workflow fails over to the `error` step and the entity transitions to `FAILED`.
- Three failure transitions land in `FAILED`: an agent error during `SEARCHING`, `SELECTING`, or `BOOKING`. A `FAILED` booking's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget, a step timeout, or a confirmation timeout transitions to `FAILED`.

There is no `CANCELLED` or `REFUNDED` state. Reservation rollback and cancellation flows are outside this blueprint's scope.

## Entity model

`BookingEntity` is the source of truth. It emits eleven event types — three lifecycle starts, four lifecycle completions, the confirmation, the guardrail audit event, the failure, and the initial creation. `BookingView` projects every event into a row used by the UI. `BookingPipelineWorkflow` both reads (`getBooking`) and writes across all steps. The relationship between `BookingAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any booking that reaches `COMMITTED`, the route passed through:

1. **Phase-gate guardrail** — every tool call is filtered. A SELECT-phase tool called during SEARCH is rejected before the tool body runs; a `GuardrailRejected` event records the violation for audit.
2. **BookingAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **User confirmation gate** — the workflow pauses at `AWAITING_CONFIRMATION`; `commitReservation` cannot execute until `BookingConfirmed` is on the entity. The guardrail enforces this independently of the workflow pause.

Each layer is independent. The phase-gate guardrail does not know about confirmation status; the confirmation gate does not know about phase ordering. Removing one of them opens an explicit gap the other does not silently cover.
