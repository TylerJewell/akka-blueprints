# Architecture — restaurant-booking-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call and one external write. `BookingEndpoint` accepts a reservation request, writes a `BookingInitiated` event onto `BookingEntity`, and starts a `BookingWorkflow` instance. The workflow's `collectStep` calls `BookingAgent` — the single AutonomousAgent — with the user's freeform request and the restaurant catalogue as task instructions. The agent's `before-tool-call` guardrail (`ReservationGuardrail`) validates each tool-call candidate before any external write fires. Once a valid `BookingProposal` returns, the workflow emits `ConfirmationRequested` and pauses in `confirmStep`.

`ConfirmationConsumer` subscribes to `ConfirmationRequested` events and tracks pending confirmations. When the user POSTs a confirm or decline, `BookingEndpoint` signals `ConfirmationConsumer`, which forwards the decision to the waiting workflow. A `CONFIRMED` decision advances to `commitStep`, which calls `BookingStub.createBooking` and writes the `BookingCompleted` event. A `DECLINED` decision or a timeout writes `BookingFailed`.

`BookingView` projects every entity event into a read-model row. `BookingEndpoint` serves this view to the UI over REST and SSE.

The graph has no second agent. `BookingStub` is a deterministic in-process stub — not an LLM. The confirm/decline decision is a human action. That is what keeps this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct pause points:

1. The `collectStep` LLM call, bounded by the 60 s step timeout.
2. The `confirmStep` wait, bounded by the 300 s (5-minute) step timeout while the user reviews the proposal.
3. The `commitStep` external stub call, bounded by 15 s.

The `before-tool-call` guardrail fires within the `collectStep` before the proposal exits the agent loop. It has no async component — it is a synchronous validator that either passes the call through or returns a structured rejection to the agent.

## State machine

Seven states. The interesting paths:

- The happy path is `INITIATED → COLLECTING → PENDING_CONFIRMATION → COMMITTING → CONFIRMED`.
- The user-decline path exits early: `PENDING_CONFIRMATION → DECLINED`. No external write occurs.
- Two failure transitions land in `FAILED`: an agent error or guardrail-exhaustion during `COLLECTING`, and a stub error during `COMMITTING`.
- The `confirmStep` 300 s timeout produces `BookingFailed("confirmation-timeout")` if the user never responds.

There is no state for "booking in dispute" or "booking amended". The blueprint deliberately stops at `CONFIRMED` or `DECLINED`. An amendment flow would require a new booking request.

## Entity model

`BookingEntity` is the source of truth. It emits eight event types. `BookingView` projects every event into a row used by the UI. `ConfirmationConsumer` subscribes to `ConfirmationRequested` events to route the user's decision to the workflow. `BookingWorkflow` both reads (`getBooking`) and writes (`startCollection`, `requestConfirmation`, `confirm`, `decline`, `startCommit`, `complete`, `fail`) on the entity. `BookingAgent`'s task result is the `BookingProposal` record.

## Governance flow

For any reservation that reaches `CONFIRMED`, the path includes:

1. **BookingAgent** — one model call extracts parameters and produces a proposal.
2. **before-tool-call guardrail** — bad dates, out-of-range party sizes, invalid emails, and after-hours times are caught before the external write.
3. **Human-in-the-loop confirmation** — the user reviews the full proposal before committing. No external write happens without an explicit `CONFIRMED` signal.

Each layer is independent. The guardrail does not know about the confirmation gate; the confirmation gate does not know about the guardrail. Removing either opens an explicit risk the other does not cover.
