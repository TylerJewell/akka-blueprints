# Architecture — it-helpdesk

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`RequestSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundRequestReceived` events into `RequestQueue` (event-sourced for audit). A `SecretSanitizer` Consumer subscribes to that queue, strips credentials from the payload, registers a `RequestEntity`, and starts a `HelpdeskWorkflow` instance.

The workflow orchestrates the handoff. It calls `ClassifierAgent` to categorize the sanitized request, then branches: `ACCESS` invokes `AccessSpecialist`, `INFRASTRUCTURE` invokes `InfraSpecialist`, `SOFTWARE` invokes `SoftwareSpecialist`, `UNCLEAR` terminates in `ESCALATED`. The chosen specialist owns the `RESOLVE` task end-to-end and returns a typed `Resolution` that may include a `ProposedTicket`. When a proposed ticket is present, the workflow calls `TicketGuardrail` to check the write against IT policy. On `allowed=true`, the ticket is filed and the resolution is published. On `allowed=false`, the request transitions to `TICKET_BLOCKED` and waits for a technician to unblock via `POST /api/requests/{id}/unblock`. When no `ProposedTicket` is present, the guardrail step is skipped.

A second Consumer, `RoutingEvalScorer`, runs independently of the workflow. It subscribes to `RequestEntity` events; on every `RequestClassified` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the access happy path). Two concurrent flows merge into `RequestEntity`:

1. The workflow path: `classifyStep` → `routeStep` → `accessStep` → `guardrailStep` → `fileStep` → `publishStep`.
2. The eval path: `RoutingEvalScorer` observes `RequestClassified` and writes `RoutingScored` in parallel.

Both write to the same `RequestEntity`; the entity's commands are idempotent on `requestId`. The view materialises both events independently, so the App UI sees a routing score appear within ~10 s of the classification decision regardless of which path completes first.

## State machine

Ten states. The interesting branches:

- After `CLASSIFIED`, the `category` value branches the flow into `ROUTED_ACCESS`, `ROUTED_INFRASTRUCTURE`, or `ROUTED_SOFTWARE`. All three converge at `RESOLUTION_DRAFTED` once the specialist returns. `UNCLEAR` jumps directly to `ESCALATED` — no specialist is invoked.
- After `RESOLUTION_DRAFTED`, the presence of a `ProposedTicket` determines whether the guardrail runs. Without one, the flow proceeds unconditionally to `RESOLVED`. With one, the guardrail verdict branches: `allowed=true` → file ticket → `RESOLVED`; `allowed=false` → `TICKET_BLOCKED`. From `TICKET_BLOCKED`, the only forward transition is technician-initiated unblock → `RESOLVED`. There is no auto-timeout on blocked requests.

`RoutingScored` events do not change `status`; the state diagram omits them as a no-op.

## Entity model

`RequestEntity` is the source of truth and emits eleven event types: registration, sanitization, classification, routing, draft, guardrail verdict, ticket filed, ticket blocked, resolution published, escalation, and routing score. `RequestQueue` is the upstream audit log — only `SecretSanitizer` subscribes to it. `RequestView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any response that is published, the request passed through:

1. **SecretSanitizer** — the model never sees raw credentials from the employee's message.
2. **ClassifierAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly misrouting ambiguous requests.
3. **Specialist agent** — owns the resolution end-to-end with a tightly-scoped prompt (authority limits, no invented links or keys, no echoed redaction tokens).
4. **TicketGuardrail** — typed before-tool-call check against the IT policy rubric. Blocks open-ended admin grants, P1 without approver, duplicate incidents, and cross-team writes without a named owner.
5. **RoutingEvalScorer** — out-of-band scoring of every routing decision.

Step 1 is preventive (credentials cannot be logged or exfiltrated via the model). Step 4 is preventive for the write path (malformed tickets are caught before they enter the provisioning queue). Step 5 is continuous (sustained low routing scores signal category drift).
