# Architecture — triage-handoff-routing

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`ConversationSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundTurnReceived` events into `ConversationQueue` (event-sourced for audit). An `IntentClassifier` Consumer subscribes to that queue, normalises the raw turn text, registers a `ConversationEntity`, and starts a `HandoffWorkflow` instance.

The workflow orchestrates the handoff in two guarded phases. The first guard fires before any specialist is called: `RoutingGuardrail` checks the triage decision for minimum confidence and intent–text alignment. A failed check transitions the turn to `ROUTING_BLOCKED` immediately — no specialist is ever invoked. A passing check proceeds to the intent branch: `ACCOUNT` invokes `AccountSpecialist`, `PRODUCT` invokes `ProductSpecialist`, `RETURNS` invokes `ReturnSpecialist`, and `UNCLEAR` terminates in `ESCALATED`. Each specialist owns the `RESOLVE` task end-to-end and returns a typed `Reply`. The second guard then fires: `ResponseGuardrail` checks the draft against the content policy. On `allowed=true`, the workflow publishes. On `allowed=false`, the turn enters `RESPONSE_BLOCKED` and sits there until an operator reviews via `POST /api/turns/{id}/review`.

## Interaction sequence

The sequence diagram traces J1 (the account happy path). Steps 7–9 show the routing guardrail running synchronously between the triage result and the specialist invocation — this is the novel structural feature of this blueprint relative to a plain handoff. The specialist is never reached unless the routing guardrail clears.

## State machine

Eleven states. The interesting branches:

- After `TRIAGED`, two branches are possible before reaching a specialist. If the routing guardrail blocks, the flow moves to `ROUTING_BLOCKED` (terminal until operator review). If the guardrail allows, the `intent` value routes to `ROUTED_ACCOUNT`, `ROUTED_PRODUCT`, or `ROUTED_RETURNS`, or directly to `ESCALATED` for `UNCLEAR` intent.
- After `REPLY_DRAFTED`, the response guardrail verdict branches again. `allowed=true` → `RESOLVED` terminal. `allowed=false` → `RESPONSE_BLOCKED`. From either blocked state, the operator's `review` command can transition to `RESOLVED` (publish) or `ESCALATED` (abandon).

There are therefore two distinct blocking surfaces — one before the specialist is ever called and one after the specialist returns. This gives operators two distinct intervention points with different urgency profiles: a routing block is cheaper to resolve (the specialist was never called) while a response block means the specialist's work is preserved and the operator only needs to decide on content.

## Entity model

`ConversationEntity` is the source of truth and emits ten event types. `ConversationQueue` is the upstream audit log — only `IntentClassifier` subscribes to it. `ConversationView` projects the entity into a single read-model row. There is no secondary Consumer (unlike the support-handoff blueprint which runs a parallel eval scorer) — the two guardrail agents are synchronous within the workflow, not out-of-band.

## Defence-in-depth governance flow

For any reply that goes out, the turn passed through:

1. **IntentClassifier** — normalisation pass that flags sensitive-data-adjacent content before any LLM sees the text.
2. **TriageAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content.
3. **RoutingGuardrail** — before-agent-invocation check on confidence and intent–text alignment. Blocks low-confidence and clearly mismatched routings before a specialist is called.
4. **Specialist agent** — owns the resolution with a tightly-scoped prompt (no invented policy citations, no out-of-scope advice, no fabricated reference numbers).
5. **ResponseGuardrail** — before-agent-response check against the content policy. Blocks invented citations, out-of-scope financial or legal advice, and out-of-policy return-window promises.

Steps 3 and 5 are both blocking guardrails; they fire at different points in the workflow lifecycle and cover different risk surfaces. Step 3 is preventive at the routing layer — it stops the wrong specialist from being called. Step 5 is preventive at the content layer — it stops a well-routed specialist from sending a policy-violating reply.
