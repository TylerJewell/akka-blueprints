# Architecture — akka-handoff-routing

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`ConversationSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundTurnReceived` events into `ConversationQueue` (event-sourced for audit). A `MessageFilter` Consumer subscribes to that queue, redacts the payload (implementing the message-filter variant of the handoff primitive), registers a `ConversationEntity`, and starts a `HandoffWorkflow` instance.

The workflow orchestrates the handoff. Its first gate is `RoutingGuardrail`: before any classification happens, the filtered context bundle is checked for unredacted identifiers, prompt-injection signals, and out-of-scope content. If the guardrail rejects, the conversation transitions to `BLOCKED` immediately — `TriageAgent` is never called and no specialist ever sees the context. If the guardrail passes, the workflow calls `TriageAgent` to classify the filtered message, then branches: `BILLING` invokes `BillingSpecialist`, `TECHNICAL` invokes `TechnicalSpecialist`, `UNCLEAR` terminates in `ESCALATED`. The chosen specialist owns the `RESOLVE` task end-to-end and returns a typed `Reply`. The workflow publishes the reply directly (the guardrail already ran before the specialist was called).

A second Consumer, `HandoffEvalScorer`, runs independently of the workflow. It subscribes to `ConversationEntity` events; on every `RoutingDecided` it invokes `HandoffJudge` and writes a `HandoffScored` event back to the same entity. This is the **on-decision eval** mechanism.

## Interaction sequence

The sequence diagram traces J1 (the billing happy path). Two distinct concurrent flows merge into `ConversationEntity`:

1. The workflow path: `filterConfirmStep` → `guardrailStep` → `triageStep` → `routeStep` → `billingStep` → `publishStep`.
2. The eval path: `HandoffEvalScorer` observes `RoutingDecided` and writes `HandoffScored` in parallel.

Both write to the same `ConversationEntity`; the entity's commands are idempotent on `conversationId`. The view materialises both events independently, so the App UI sees a handoff score appear within ~10 s of the routing decision.

## State machine

Nine states. The key structural difference from a response-guardrail pattern is that `BLOCKED` is reachable from `FILTERED` — not from a later drafted state. This means:

- After `FILTERED`, the guardrail verdict branches immediately. `allowed=true` → `GUARDRAIL_PASSED`; `allowed=false` → `BLOCKED`. No classifier or specialist is invoked when blocked.
- After `GUARDRAIL_PASSED`, the category value branches the flow. `BILLING` and `TECHNICAL` go to their respective `ROUTED_*` state, then converge at `REPLY_READY` once the specialist returns. `UNCLEAR` goes straight to the `ESCALATED` terminal.
- From `REPLY_READY`, the workflow emits `ReplyPublished` → `RESOLVED` terminal.
- From `BLOCKED`, the only forward transition is operator-initiated unblock → `RESOLVED`.

`HandoffScored` events do not change `status`; they attach the eval score to the entity.

## Entity model

`ConversationEntity` is the source of truth and emits ten event types covering registration, filtering, guardrail verdict, routing decision, routing assignment, draft, publish, block, escalate, and the eval score. `ConversationQueue` is the upstream audit log — only `MessageFilter` subscribes to it. `ConversationView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any reply that goes out to the customer, the conversation passed through:

1. **MessageFilter** — the message-filter variant of the handoff primitive; no LLM ever sees raw identifiers.
2. **RoutingGuardrail** — typed before-agent-invocation check that blocks poisoned contexts before any LLM decision is made.
3. **TriageAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content.
4. **Specialist agent** — owns the resolution end-to-end with a tightly-scoped prompt (refund authority limit, no-medical-advice, no-invented-articles).
5. **HandoffEvalScorer** — out-of-band scoring of every routing decision.

Step 1 is a data-boundary control (the model can't leak what it never sees). Step 2 is a pre-invocation gate (a poisoned context never reaches a specialist). Step 5 is a continuous signal (a sustained drop in scores would indicate a regression in the classifier or the filter's coverage).
