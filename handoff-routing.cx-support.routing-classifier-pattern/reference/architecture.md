# Architecture — routing-classifier-pattern

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`MessageSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes `InboundMessageReceived` events into `MessageQueue` (event-sourced for audit). A `MessageIngestor` Consumer subscribes to that queue, registers a `MessageEntity`, and starts a `RoutingWorkflow` instance per message.

The workflow orchestrates a two-stage governance gate followed by a specialist handoff. It first calls `RoutingClassifier` to produce a `RouteDecision`, then immediately calls `RouteGuardrail` to validate that decision against the allowed-route registry and the minimum confidence floor — before any specialist is invoked. Only if the route is approved does the workflow branch to the matching specialist (`GeneralAgent`, `RefundAgent`, or `TechnicalAgent`). An `UNROUTABLE` decision routes to `abandonStep` rather than to the guardrail; the guardrail exists to catch classifier outputs that are structurally valid but semantically wrong (route name not in registry, or too-low confidence on a nominally routable category).

Once the chosen specialist returns a `Reply`, the workflow calls `ReplyGuardrail` to screen the draft. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `REPLY_BLOCKED` and the message sits there until an operator unblocks via `POST /api/messages/{id}/unblock`.

There is no out-of-band eval consumer in this baseline. The two guardrails are synchronous, in-workflow gates.

## Interaction sequence

The sequence diagram traces J1 (the general-enquiry happy path). The two distinct governance positions are visible as discrete steps:

1. `classifyStep` → `RoutingClassifier` → `RouteDecision`.
2. `validateRouteStep` → `RouteGuardrail` → `RouteVerdict`. This is the **before-agent-invocation** gate. The specialist is not called until this step completes with `approved=true`.
3. `generalStep` (or `refundStep` / `technicalStep`) → specialist → `Reply`.
4. `screenStep` → `ReplyGuardrail` → `ReplyVerdict`. This is the **before-agent-response** gate. The reply is not published until this step completes with `allowed=true`.
5. `publishStep` → `ReplyPublished`.

## State machine

Ten states. The interesting branches:

- After `CLASSIFIED`, the `RouteVerdict` result forks: `approved=false` → `ROUTE_BLOCKED` (terminal, no forward path); `approved=true` branches on `route`. `UNROUTABLE` → `ABANDONED` (terminal). `GENERAL` / `REFUND` / `TECHNICAL` → their respective `ROUTED_*` state.
- After `REPLY_DRAFTED`, the `ReplyVerdict` result forks: `allowed=true` → `PUBLISHED` (terminal). `allowed=false` → `REPLY_BLOCKED`. From `REPLY_BLOCKED`, the only forward transition is operator unblock → `PUBLISHED`.

The difference between `ROUTE_BLOCKED` and `REPLY_BLOCKED` reflects the two distinct failure modes: `ROUTE_BLOCKED` means the classifier produced a decision that the system refused to act on (no specialist involvement); `REPLY_BLOCKED` means a specialist produced a draft that the system refused to send (specialist was invoked, draft exists in entity state, operator can review).

## Entity model

`MessageEntity` emits nine event types covering registration, classification, route blocking, routing, draft, reply verdict, publish, block, and abandon. `MessageQueue` is the upstream audit log — only `MessageIngestor` subscribes to it. `MessageView` projects the entity into a single read-model row. There is no separate eval-scoring consumer; the two guardrail verdicts are the primary accountability signals.

## Defence-in-depth governance flow

For any reply that goes out to a customer, the message passed through:

1. **RoutingClassifier** — typed classifier with a default-to-`UNROUTABLE` policy that prevents quietly mis-routing ambiguous content.
2. **RouteGuardrail** — before-agent-invocation check against the allowed-route registry and confidence floor. Rejects structurally invalid route decisions before any specialist sees the message.
3. **Specialist agent** — owns the reply end-to-end with a tightly-scoped prompt (authority limits, no-fabrication rules, no cross-domain claims).
4. **ReplyGuardrail** — before-agent-response check against the content-policy rubric. Blocks invented dates, invented amounts, unverifiable legal claims, and fabricated article ids.

Steps 2 and 4 are both blocking. A message can be stopped at either gate. Step 2 stopping a message is invisible to the specialists (no specialist work is wasted). Step 4 stopping a draft surfaces to the operator with the full draft and violation list for review.
