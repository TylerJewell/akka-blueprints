# Architecture — line-personal-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`LineWebhookEndpoint` is the entry point for all LINE platform callbacks. An inbound message is written as a `MessageEnqueued` event to `MessageQueue` (event-sourced for audit). `MessageConsumer` subscribes to that queue and starts a `ConversationWorkflow` per inbound message. The workflow drives the planner-executor loop: it asks `PlannerAgent` to interpret the message and produce an `ActionPlan`, then routes to either `CalendarExecutorAgent` or `GmailExecutorAgent`. Every transition emits an event on `ConversationEntity`; `ConversationView` projects those events into the read model the monitoring UI streams via SSE.

`ConfirmationEntity` is a short-lived entity, one instance per pending email confirmation. When an outbound email needs user approval, the workflow opens a `ConfirmationEntity`, sends the confirmation message to LINE, and parks in `AWAITING_CONFIRMATION`. The LINE platform delivers the user's YES/NO reply as a new webhook event, which `LineWebhookEndpoint` routes to `ConfirmationEntity.resolveConfirmation`.

`MessageSimulator` drips sample messages every 90 s for demo purposes. `StaleConversationMonitor` ticks every 60 s to expire confirmations and clarification requests that have gone unanswered past 10 minutes.

## Interaction sequence

The sequence diagram traces the combined happy path for both the Calendar and email-with-confirmation flows. The branch at step 8 is the key divergence: calendar intents proceed directly to `executeStep`, while email intents enter the confirm-gate detour. The diagram shows a YES reply; the NO and EXPIRED branches end in `CANCELLED` and `EXPIRED` respectively.

## State machine

`ConversationEntity` has nine states. `PLANNING` is the initial state. From there the conversation branches based on guardrail outcome and intent: calendar intents go `PLANNING → EXECUTING → COMPLETED`; email intents with confirmation go `PLANNING → AWAITING_CONFIRMATION → EXECUTING → COMPLETED`; blocked plans go `PLANNING → BLOCKED`; denied or expired confirmations go `AWAITING_CONFIRMATION → CANCELLED` or `AWAITING_CONFIRMATION → EXPIRED`. `AWAITING_CLARIFICATION` is used when the planner determines it needs more information from the user. All terminal states are leaves — no recovery transitions.

## Entity model

`ConversationEntity` is the source of truth for conversation lifecycle and emits eleven event types. `ConfirmationEntity` is a narrow satellite entity whose sole job is to track the user's YES/NO response during the email confirmation window. `MessageQueue` is the audit log of every inbound LINE webhook event — conversations are started from this log, not from the webhook directly. `ConversationView` is the only read-side projection; the monitoring UI never queries entities directly.

## Concurrency notes

- Per-step timeouts: `planStep` 45 s, `executeStep` 60 s, `confirmRequestStep` 20 s, `replyStep` 20 s.
- Confirmation window: `awaitConfirmStep` polls `ConfirmationEntity.get` every 3 s; `StaleConversationMonitor` independently expires the entity at 10 minutes.
- Guardrail determinism: `ActionPlanGuardrail.vet` is a pure function; it never inspects external state. The same `ActionPlan` always produces the same verdict.
- LINE webhook idempotency: `LineWebhookEndpoint` deduplicates on the `X-Line-Message-Id` header; duplicate deliveries from the LINE platform are dropped.
- Stuck detection: `StaleConversationMonitor` ticks every 60 s; conversations in `AWAITING_CONFIRMATION` or `AWAITING_CLARIFICATION` older than 10 minutes are expired.
