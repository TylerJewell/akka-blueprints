# Architecture — support-multi-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`TicketSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundCaseReceived` events into `TicketQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, registers a `CaseEntity`, and starts a `SupportWorkflow` instance.

The workflow orchestrates the handoff. It calls `RouterAgent` to classify the sanitized request, then branches: `BILLING` invokes `BillingSpecialist`, `TECHNICAL` invokes `TechnicalSpecialist`, `ACCOUNT` invokes `AccountSpecialist`, `UNCLEAR` terminates in `ESCALATED`. The chosen specialist owns the `RESOLVE` task end-to-end and returns a typed `Resolution`. The workflow then calls `DraftGuardrail` to check the draft against the policy rubric. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the case sits there until an operator unblocks via `POST /api/cases/{id}/unblock`.

## Interaction sequence

The sequence diagram traces J1 (the billing happy path). The `PiiSanitizer` Consumer runs before the workflow starts; the workflow's first real step is the routing call.

1. `TicketSimulator` → `TicketQueue` → `PiiSanitizer` → `CaseEntity` (register + sanitize).
2. `PiiSanitizer` starts `SupportWorkflow` with `caseId`.
3. `routeStep` → `RouterAgent` → `RoutingDecision{BILLING}` → `CaseEntity.recordRouting`.
4. `billingStep` → `BillingSpecialist` → `Resolution` → `CaseEntity.recordDraft`.
5. `guardrailStep` → `DraftGuardrail` → `GuardrailVerdict{allowed=true}` → `CaseEntity.publish` → `RESOLVED`.

## State machine

Nine states. The interesting branches:

- After `SANITIZED`, the `category` value branches the flow into one of four paths: `BILLING`, `TECHNICAL`, and `ACCOUNT` each go to their respective `ROUTED_*` state; `UNCLEAR` jumps straight to the `ESCALATED` terminal — no specialist is invoked at all.
- After `RESOLUTION_DRAFTED`, the guardrail verdict branches again. `allowed=true` → `RESOLVED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `RESOLVED`. There is no auto-timeout — blocked drafts wait indefinitely.

## Entity model

`CaseEntity` is the source of truth and emits eight event types covering registration, sanitization, routing, draft, guardrail verdict, publish, block, and escalation. `TicketQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `CaseView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any reply that goes out, the case passed through:

1. **PII sanitizer** — the model never sees raw identifiers.
2. **RouterAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content.
3. **Specialist agent** — owns the resolution end-to-end with a tightly-scoped prompt (refund authority limit, no-medical-advice, no-invented-articles, no-echoed-tokens).
4. **DraftGuardrail** — typed before-agent-response check against the policy rubric. Blocks invented refund amounts, invented timelines, scope-violations, and `[REDACTED]` echoes.

Step 1 is preventive (the model cannot leak what it never sees). Step 4 is detective (drafts that violate get held). A sustained pattern of guardrail blocks across one specialist would indicate a prompt or routing issue requiring human review.
