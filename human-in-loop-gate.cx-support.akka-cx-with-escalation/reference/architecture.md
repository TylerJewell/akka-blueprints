# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a durable conversation workflow in the human-in-loop-gate pattern: receive, reply, await escalation decision, optionally escalate.

## Component graph

`SupportEndpoint` accepts an inbound customer message and starts `SupportWorkflow` with a fresh conversation id. Before the workflow starts, a PII sanitizer strips recognized personal data from the message. The workflow drives `SupportAgent` to produce a reply and an action decision, writes the reply to `ConversationEntity`, then polls in `awaitEscalationDecisionStep`. If the customer or the agent's action triggers escalation, the conversation moves to `ESCALATING`. A human agent calls accept-escalation through the endpoint, which sets `acceptedBy` on `ConversationEntity`. On the next poll the workflow transitions to `acceptEscalationStep`, drives `EscalationAgent`, and records the handoff summary. If the conversation resolves directly, the workflow ends without calling `EscalationAgent`. `ConversationEntity` events project into `ConversationsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary escalation path runs message → reply → pause → escalation request → human acceptance → handoff. The `awaitEscalationDecisionStep` does not hold a thread: it reads `ConversationEntity.getConversation`, and while the status is `ACTIVE` it self-schedules a 5-second resume timer. When the conversation moves to `ESCALATING`, the next poll transitions to `acceptEscalationStep`. That step similarly polls until `acceptedBy` is present, then calls `EscalationAgent`. A `RESOLVED` status ends the workflow without escalation.

## State machine

A conversation moves `ACTIVE → ESCALATING → ESCALATED` on the escalation path, or `ACTIVE → RESOLVED` when the agent handles the issue directly. Both `ESCALATED` and `RESOLVED` are terminal states reached only through their corresponding events (`EscalationAccepted` + `recordHandoff`, and `ConversationResolved`).

## Entity model

`ConversationEntity` is event-sourced; each command emits one event that the applier folds into the `Conversation` state. `ConversationsView` projects the same events into a row keyed by conversation id. There is one view query, `getAllConversations`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2). The PII sanitizer runs upstream of the entity, so no raw personal data appears in the event log.
