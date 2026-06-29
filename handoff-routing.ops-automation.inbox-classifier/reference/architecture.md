# Architecture — inbox-classifier

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`InboxSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundMessageReceived` events into `InboxQueue` (event-sourced for audit). A `BodySanitizer` Consumer subscribes to that queue, redacts PII from the body, registers a `MessageEntity`, and starts a `RoutingWorkflow` instance.

The workflow orchestrates the handoff. It calls `ClassifierAgent` to label the sanitized message, then calls `RoutingAgent` with the `ROUTE` task. The routing agent owns the action decision end-to-end and returns a typed `RoutingDecision`. For destructive actions (`MOVE_TO_SPAM`, `DELETE`), the workflow calls `ActionGuardrail` to verify the action is safe before executing. On `allowed=true`, the workflow emits `ActionExecuted` (terminal `ACTIONED`). On `allowed=false`, it emits `ActionBlocked` (terminal `BLOCKED`) and the message sits there until an operator unblocks via `POST /api/messages/{id}/unblock`.

A second Consumer, `ClassificationEvalScorer`, runs independently of the workflow. It subscribes to `MessageEntity` events; on every `MessageClassified` it invokes `ClassificationJudge` and writes a `ClassificationScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the urgent message happy path). Two concurrent flows merge into `MessageEntity`:

1. The workflow path: `classifyStep` → `routeStep` → `executeStep` (guardrail skipped for non-destructive `FLAG_URGENT`).
2. The eval path: `ClassificationEvalScorer` observes `MessageClassified` and writes `ClassificationScored` in parallel.

Both write to the same `MessageEntity`; commands are idempotent on `messageId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a classification score appear within ~10 s of the label decision regardless of which path completes first.

Note that for `FLAG_URGENT` and other non-destructive actions, the guardrail step is skipped entirely. The workflow's `executeStep` fires directly, which keeps the critical path short for the high-frequency case.

## State machine

Eight states. The distinctive branches:

- After `CLASSIFIED`, the workflow emits `RoutingDecisionRecorded` and moves to `ROUTING_DECIDED`. From there, the decision forks: destructive actions transition to `GUARDRAIL_PENDING`; non-destructive actions skip directly to `ACTIONED`.
- From `GUARDRAIL_PENDING`, the guardrail verdict determines the outcome. `allowed=true` → `ACTIONED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `ACTIONED`. There is no auto-timeout.
- `ESCALATED` is the terminal for workflow errors (step timeout after max retries). It is not a routine routing outcome.

`ClassificationScored` events do not change `status`; they attach the eval score. The state diagram omits this as a no-op for clarity.

## Entity model

`MessageEntity` is the source of truth and emits nine event types covering registration, sanitization, classification, routing, guardrail verdict, action execution, action block, escalation, and the eval score. `InboxQueue` is the upstream audit log — only `BodySanitizer` subscribes to it. `MessageView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any message that is actioned, it passed through:

1. **PII sanitizer** — the model never sees raw email addresses, phone numbers, or account identifiers.
2. **ClassifierAgent** — typed classifier with a default-to-`INFO` policy that prevents an ambiguous message from being silently routed to a destructive action.
3. **RoutingAgent** — owns the action decision with a tightly-scoped prompt (restricted folder list, narrow `DELETE` criteria, conservative tie-breaking).
4. **ActionGuardrail** — typed before-tool-call check on every destructive action. Blocks unwarranted deletes, classification-contradicting spam moves, and low-confidence permanent deletions.
5. **ClassificationEvalScorer** — out-of-band scoring of every classification decision, surfaced as a per-message chip in the UI.

Step 1 is preventive (the model cannot leak what it never sees). Step 4 is a last-resort gate on irreversible operations. Step 5 is a continuous quality signal that would feed a system-level alert on sustained regression.
