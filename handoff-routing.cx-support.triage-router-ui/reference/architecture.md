# Architecture — triage-router-ui

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`MessageSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundMessageReceived` events into `MessageQueue` (event-sourced for audit). For each new message, the simulator also starts a `TriageWorkflow` instance keyed by `messageId`.

The workflow orchestrates the handoff. It calls `RouterAgent` to classify the incoming message, then branches: `BILLING` invokes `BillingHandler`, `PRODUCT` invokes `ProductHandler`, `UNCLEAR` terminates in `ESCALATED`. The chosen handler owns the `RESOLVE` task end-to-end and returns a typed `HandlerReply`. The workflow then calls `DraftGuardrail` to check the draft against the policy rubric. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the message sits there until an operator unblocks via `POST /api/messages/{id}/unblock`.

This baseline has no Consumer between the queue and the workflow — the simulator starts the workflow directly. There is also no on-decision eval Consumer; the single governance mechanism is the before-agent-response guardrail.

## Interaction sequence

The sequence diagram traces J1 (the billing happy path). The flow is strictly linear: `routeStep` → `billingStep` → `guardrailStep` → `publishStep`. There is no concurrent eval path in this baseline.

The `DraftGuardrail` check at step 9 is the gate. If `allowed=false`, the workflow emits `ReplyBlocked` and the UI shows the blocked draft along with the violation list and an Unblock button. The draft is never marked as published until either the guardrail passes or an operator explicitly overrides.

## State machine

Seven states. The interesting branches:

- After `RECEIVED`, the `category` value from `RouterAgent` immediately branches the flow. `BILLING` and `PRODUCT` go to their respective `ROUTED_*` states, then converge at `REPLY_DRAFTED` once the handler returns. `UNCLEAR` jumps straight to the `ESCALATED` terminal — no handler is invoked at all.
- After `REPLY_DRAFTED`, the guardrail verdict branches. `allowed=true` → `RESOLVED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `RESOLVED`.

The `RECEIVED` → `ROUTED_*` transition is tight: there is no `SANITIZED` intermediate state in this baseline (no PII sanitizer Consumer), so routing happens as the first workflow step.

## Entity model

`MessageEntity` is the source of truth and emits seven event types covering registration, routing, draft, guardrail result, publish, block, and escalation. `MessageQueue` is the upstream audit log — its events trigger workflow start rather than feeding a Consumer. `MessageView` projects the entity into a single read-model row per message.

## Governance flow

For any reply that goes out, the message passed through:

1. **RouterAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content to the wrong handler's authority set.
2. **Handler agent** — owns the resolution end-to-end with a tightly-scoped prompt (no invented amounts, no invented timelines, no out-of-scope advice, no invented article ids).
3. **DraftGuardrail** — typed before-agent-response check against the policy rubric. Blocks invented timelines, out-of-scope advice, echoed redacted tokens, and invented article ids.

Step 3 is the formal governance control declared in `eval-matrix.yaml`. Steps 1 and 2 are prompt-level constraints that reduce the probability of guardrail trips rather than constituting a separate enforcement mechanism.
