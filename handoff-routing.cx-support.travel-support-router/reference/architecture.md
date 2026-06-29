# Architecture — travel-support-router

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`RequestSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundRequestReceived` events into `RequestQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, registers a `RequestEntity`, and starts a `TravelWorkflow` instance.

The workflow orchestrates the handoff. It calls `RouterAgent` to classify the sanitized request, then branches: `FLIGHTS` invokes `FlightSpecialist`, `HOTELS` invokes `HotelSpecialist`, `CAR_RENTAL` invokes `CarRentalSpecialist`, `EXCURSIONS` invokes `ExcursionSpecialist`, and `UNCLEAR` terminates in `ESCALATED`. Before any booking mutation executes, the workflow calls `BookingGuardrail` to check the proposed action against the travel-policy rubric. On `allowed=false`, the request goes to `BLOCKED` for operator review. On `allowed=true`, the workflow calls `ConfirmationGate` which produces a plain-language change summary. The workflow then pauses, waiting for the passenger to `CONFIRM` or `REJECT` via `POST /api/requests/{id}/confirm`. A `CONFIRM` resumes the workflow and publishes the resolution; a `REJECT` terminates in `PASSENGER_REJECTED` with no mutation.

A second Consumer, `RouterEvalScorer`, runs independently of the workflow. It subscribes to `RequestEntity` events; on every `RoutingDecided` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the flight rebooking happy path). Three concurrent concerns converge on `RequestEntity`:

1. **The workflow path:** `routeStep` → `specialistStep` → `guardrailStep` → `confirmStep` → `publishStep`.
2. **The eval path:** `RouterEvalScorer` observes `RoutingDecided` and writes `RoutingScored` in parallel, without affecting the workflow.
3. **The passenger path:** the workflow pauses at `confirmStep`; the passenger's `POST /confirm` resumes it.

All three write to the same `RequestEntity`; commands are idempotent on `requestId` and mutate disjoint fields, so there is no conflict.

## State machine

Fourteen states. The interesting branches:

- After `ROUTED`, the `category` value fans out to `ROUTED_FLIGHTS`, `ROUTED_HOTELS`, `ROUTED_CAR_RENTAL`, or `ROUTED_EXCURSIONS` — or straight to `ESCALATED` on `UNCLEAR`.
- All four `ROUTED_*` states funnel through the guardrail. On `allowed=true` → `GUARDRAIL_CLEARED`. On `allowed=false` → `BLOCKED` (terminal until operator unblock).
- From `GUARDRAIL_CLEARED` → `AWAITING_CONFIRMATION` when `ConfirmationRequested` is emitted. From there, `PassengerConfirmed` → `RESOLUTION_DRAFTED` → `RESOLVED`, or `PassengerRejected` → `PASSENGER_REJECTED` (terminal, no mutation).

`RoutingScored` events do not change `status`; the state diagram omits them.

## Entity model

`RequestEntity` is the source of truth and emits thirteen event types. `RequestQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `RequestView` projects `RequestEntity` into a single read-model row.

## Defence-in-depth governance flow

For any booking change that goes out, the request passed through:

1. **PII sanitizer** — the model never sees raw travel-document or financial identifiers.
2. **RouterAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content.
3. **Specialist agent** — owns the resolution end-to-end with a tightly-scoped prompt (authority limits, no-invented-details, no-token-echo).
4. **BookingGuardrail** — typed before-tool-call check against the travel-policy rubric. Blocks non-refundable cancellations outside the window, upgrades above entitlement, insurance-prompt bypasses, and unauthorized compensation.
5. **ConfirmationGate** — application HITL that produces a plain-language change summary and pauses until the passenger explicitly confirms. A `REJECT` terminates the flow with no mutation.
6. **RouterEvalScorer** — out-of-band scoring of every routing decision.

Step 1 is preventive (the model cannot leak what it never sees). Step 4 is detective (policy-violating tool calls are blocked before execution). Step 5 is confirmatory (the passenger has final say before any mutation). Step 6 is continuous (a sustained drop in routing scores would signal a regression in the classifier).
