# Architecture — discord-router

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`MessageSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundMessageReceived` events into `MessageQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, registers a `MessageEntity`, and starts a `DiscordWorkflow` instance.

The workflow orchestrates the handoff. It calls `ClassifierAgent` to classify the sanitized message, then branches: `COMMUNITY` invokes `CommunitySpecialist`, `TECHNICAL` invokes `TechnicalSpecialist`, `UNCLEAR` terminates in `ESCALATED`. The chosen specialist owns the `REPLY` task end-to-end and returns a typed `BotReply`. The workflow then calls `ReplyGuardrail` to check the draft against the community-policy rubric. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the message sits there until a moderator unblocks via `POST /api/messages/{id}/unblock`.

A second Consumer, `RoutingEvalScorer`, runs independently of the workflow. It subscribes to `MessageEntity` events; on every `MessageClassified` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the community happy path). Two distinct concurrent flows merge into `MessageEntity`:

1. The workflow path: `classifyStep` → `routeStep` → `communityStep` → `guardrailStep` → `publishStep`.
2. The eval path: `RoutingEvalScorer` observes `MessageClassified` and writes `RoutingScored` in parallel.

Both write to the same `MessageEntity`; the entity's commands are idempotent on `messageId` and the events do not collide. The view materialises both events independently, so the App UI sees a routing score appear within ~10 s of the classification decision regardless of which path completes first.

## State machine

Nine states. The interesting branches:

- After `CLASSIFIED`, the `category` value branches the flow. `COMMUNITY` and `TECHNICAL` go to their respective `ROUTED_*` state, then converge at `REPLY_DRAFTED` once the specialist returns. `UNCLEAR` jumps straight to the `ESCALATED` terminal — no specialist is invoked.
- After `REPLY_DRAFTED`, the guardrail verdict branches again. `allowed=true` → `PUBLISHED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is moderator-initiated unblock → `PUBLISHED`. Blocked messages wait indefinitely; there is no auto-timeout.

`RoutingScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this for clarity.

## Entity model

`MessageEntity` is the source of truth and emits ten event types covering registration, sanitization, classification, routing, draft, guardrail verdict, publish, block, escalation, and the eval score. `MessageQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `MessageView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any reply that goes out, the message passed through:

1. **PII sanitizer** — the model never sees raw handles, user ids, or contact details.
2. **ClassifierAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content.
3. **Specialist agent** — owns the reply end-to-end with a tightly-scoped prompt (no invented URLs, no staff impersonation, no off-topic promotion).
4. **ReplyGuardrail** — typed before-agent-response check against the community-policy rubric. Blocks unapproved external links, staff impersonation, `[REDACTED]` echoes, and off-topic promotion.
5. **RoutingEvalScorer** — out-of-band scoring of every routing decision.

Step 1 is preventive. Step 4 is detective. Step 5 is continuous — a sustained drop in routing scores would signal a regression in the classifier prompt or a distribution shift in incoming messages.
