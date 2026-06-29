# Architecture — support

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`RequestSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundRequestReceived` events into `RequestQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, registers a `TicketEntity`, and starts a `SupportWorkflow` instance.

The workflow orchestrates the handoff. It calls `TriageAgent` to classify the sanitized request, then branches: `BILLING` invokes `BillingSpecialist`, `TECHNICAL` invokes `TechnicalSpecialist`, `UNCLEAR` terminates in `ESCALATED`. The chosen specialist owns the `RESOLVE` task end-to-end and returns a typed `Resolution`. The workflow then calls `ResponseGuardrail` to check the draft against the policy rubric. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the ticket sits there until an operator unblocks via `POST /api/tickets/{id}/unblock`.

A second Consumer, `TriageEvalScorer`, runs independently of the workflow. It subscribes to `TicketEntity` events; on every `TriageDecided` it invokes `TriageJudge` and writes a `TriageScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the billing happy path). Two distinct concurrent flows merge into `TicketEntity`:

1. The workflow path: `triageStep` → `routeStep` → `billingStep` → `guardrailStep` → `publishStep`.
2. The eval path: `TriageEvalScorer` observes `TriageDecided` and writes `TriageScored` in parallel.

Both write to the same `TicketEntity`; the entity's commands are idempotent on `ticketId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a triage score appear within ~10 s of the routing decision regardless of which path completes first.

## State machine

Nine states. The interesting branches:

- After `TRIAGED`, the `category` value branches the flow. `BILLING` and `TECHNICAL` go to their respective `ROUTED_*` state, then converge at `RESOLUTION_DRAFTED` once the specialist returns. `UNCLEAR` jumps straight to the `ESCALATED` terminal — no specialist is invoked at all.
- After `RESOLUTION_DRAFTED`, the guardrail verdict branches again. `allowed=true` → `RESOLVED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `RESOLVED`. There is no auto-timeout — blocked drafts wait indefinitely.

`TriageScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this as a no-op for clarity.

## Entity model

`TicketEntity` is the source of truth and emits ten event types covering registration, sanitization, triage, routing, draft, guardrail verdict, publish, block, escalate, and the eval score. `RequestQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `TicketView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any reply that goes out, the ticket passed through:

1. **PII sanitizer** — the model never sees raw identifiers.
2. **TriageAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content.
3. **Specialist agent** — owns the resolution end-to-end with a tightly-scoped prompt (refund authority limit, no-medical-advice, no-invented-articles).
4. **ResponseGuardrail** — typed before-agent-response check against the policy rubric. Blocks invented refund amounts, invented timelines, scope-violations, and `[REDACTED]` echoes.
5. **TriageEvalScorer** — out-of-band scoring of every routing decision.

Step 1 is preventive (the model can't leak what it never sees). Step 4 is detective (drafts that violate get held). Step 5 is continuous (a sustained drop in triage scores would signal a regression).
