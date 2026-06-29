# Architecture — routing-classifier

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`TicketSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundTicketReceived` events into `TicketQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, registers a `CaseEntity`, and starts a `RoutingWorkflow` instance.

The workflow orchestrates the routing. It calls `ClassifierAgent` to categorize the sanitized ticket, then branches: `BILLING` invokes `BillingHandler`, `TECHNICAL` invokes `TechnicalHandler`, `ACCOUNT` invokes `AccountHandler`, `UNCLEAR` terminates in `ESCALATED`. The chosen handler owns the `RESPOND` task end-to-end and returns a typed `HandlerReply`. The workflow then calls `ResponseGuardrail` to check the draft against the policy rubric. On `allowed=true`, the workflow publishes the reply; on `allowed=false`, it transitions the case to `BLOCKED` and the case sits there until an operator unblocks via `POST /api/cases/{id}/unblock`.

Unlike the two-specialist variant of this pattern, the three-way branch (`BILLING` / `TECHNICAL` / `ACCOUNT`) means each handler's scope is narrower — which raises the value of the classifier's precision and of the guardrail's category-mismatch rule.

## Interaction sequence

The sequence diagram traces J1 (the billing happy path). The flow is linear: `classifyStep` → `routeStep` → `billingStep` → `guardrailStep` → `publishStep`. Each step writes one or more events to `CaseEntity`; the entity projects into `CaseView` which the UI reads via SSE.

The key difference from a single-step routing pattern: the classifier's output (`ClassificationResult`) is first recorded on the entity before the branch executes. This means the classification is auditable even if the downstream handler fails.

## State machine

Ten states. The interesting branches:

- After `CLASSIFIED`, the `category` value splits the flow into three `ROUTED_*` states (or `ESCALATED` for `UNCLEAR`). All three routed states converge at `REPLY_DRAFTED` once the handler returns.
- After `REPLY_DRAFTED`, the guardrail verdict branches again: `allowed=true` → `RESOLVED` terminal; `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `RESOLVED`.

Because there is no eval-event mechanism in this blueprint, there is no auxiliary state-mutating event (unlike the eval-scored pattern). Every state transition is caused by the main workflow or by an operator action.

## Entity model

`CaseEntity` is the source of truth and emits nine event types covering registration, sanitization, classification, routing, draft, guardrail verdict, publish, block, and escalation. `TicketQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `CaseView` projects the entity into a single read-model row that the endpoint queries and streams over SSE.

## Governance flow

For any reply that reaches the customer, the case passed through:

1. **PII sanitizer** — the model never sees raw contact details or account identifiers.
2. **ClassifierAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content.
3. **Handler agent** — owns the resolution end-to-end with a tightly-scoped prompt (authority limits, no-fabrication rules, no-scope-violation).
4. **ResponseGuardrail** — typed before-agent-response check against the policy rubric. Blocks invented timelines, out-of-scope advice, `[REDACTED]` echoes, and invented knowledge-base article ids.

Step 1 is preventive (the model cannot leak what it never sees). Step 4 is a gate (drafts that violate are held for operator review, not silently dropped or auto-retried). Together they give the deployer a two-point governance boundary around the LLM layer.
