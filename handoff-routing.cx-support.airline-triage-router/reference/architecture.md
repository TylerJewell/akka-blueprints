# Architecture — airline-triage-router

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`RequestFeeder` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundPassengerRequestReceived` events into `PassengerRequestQueue` (event-sourced for audit before any redaction occurs). A `PnrSanitizer` Consumer subscribes to that queue, strips PNR locators, passport numbers, frequent-flyer numbers, and email addresses from the payload, registers a `FlightRequestEntity`, and starts a `FlightRequestWorkflow` instance.

The workflow orchestrates the handoff. It calls `IntentRouter` to classify the sanitized request into one of five intents, then branches: `BOOKING` invokes `BookingSpecialist`, `CHANGE` invokes `ChangeSpecialist`, `BAGGAGE` invokes `BaggageSpecialist`, `STATUS` invokes `StatusSpecialist`, and `UNCLEAR` terminates in `UNRESOLVED`. The chosen specialist owns the `RESOLVE` task end-to-end. Before any destructive tool action, the specialist invokes `ToolCallGuardrail`; on denial the request moves to `BLOCKED`. Once the specialist returns a draft, the workflow invokes `ResponseGuardrail`; on `allowed=true` the response is published, on `allowed=false` the request moves to `BLOCKED`.

A second Consumer, `RoutingEvalScorer`, runs independently of the workflow. It subscribes to `FlightRequestEntity` events; on every `IntentClassified` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the booking happy path). Two concurrent flows merge into `FlightRequestEntity`:

1. The workflow path: `classifyStep` → `routeStep` → `bookingStep` → (tool-call guardrail inline) → `responseGuardrailStep` → `publishStep`.
2. The eval path: `RoutingEvalScorer` observes `IntentClassified` and writes `RoutingScored` in parallel.

Both write to the same `FlightRequestEntity`; the entity's commands are idempotent on `requestId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a routing score appear within ~10 s of the classification regardless of which path completes first.

## State machine

Eleven states. The branching structure:

- After `CLASSIFIED`, the `intent` value branches the flow into four `ROUTED_*` states or the `UNRESOLVED` terminal. All four routed states converge at `RESOLUTION_DRAFTED` once the specialist returns a draft.
- From any `ROUTED_*` state, a tool-call guardrail denial can jump directly to `BLOCKED` — before the draft is produced. This is distinct from the post-draft `BLOCKED` state, but both share the same entity state value; the operator unblocks via the same endpoint.
- After `RESOLUTION_DRAFTED`, the response guardrail verdict branches to `RESOLVED` (terminal) or `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `RESOLVED`.

`RoutingScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this as a no-op transition.

## Entity model

`FlightRequestEntity` is the source of truth and emits eleven event types covering registration, sanitization, classification, routing, draft, tool-call verdict, response verdict, publish, block, unresolved, and the eval score. `PassengerRequestQueue` is the upstream audit log — only `PnrSanitizer` subscribes to it. `FlightRequestView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any passenger reply that goes out, the request passed through:

1. **PNR sanitizer** — no LLM ever sees a raw booking reference, passport number, or frequent-flyer number.
2. **IntentRouter** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous requests.
3. **ToolCallGuardrail** — checked before every destructive tool action inside the specialist. Non-refundable cancellations, out-of-window changes, and authority-limit violations are blocked before they touch the booking system.
4. **Specialist agent** — owns the resolution end-to-end with a tightly-scoped prompt (no invented amounts, no invented timelines, no legal advice).
5. **ResponseGuardrail** — typed before-agent-response check against the response rubric. Blocks PNR echoes, invented compensation amounts, invented delivery timelines, and scope violations.
6. **RoutingEvalScorer** — out-of-band scoring of every intent classification.

Step 1 is preventive (models can't leak what they never see). Step 3 is preventive at the action boundary (destructive operations are checked before execution). Step 5 is detective (drafts that violate are held). Step 6 is continuous (a sustained drop in routing scores signals a classifier regression).
