# Architecture

The generated system models a monitored support inbox end to end inside one Akka service. Nothing external is required to run it: the inbound mail source and the outbound send action are both built from the same primitives used elsewhere in the service.

The four diagrams below are the mermaid sources the Architecture tab renders. They carry the Akka theme variables and, for the state machine, the Lesson 24 label-colour fixes.

## Component graph

A `InboxMonitor` TimedAction continuously samples the canned inbound source and drips one email at a time into the `InboxQueue` event-sourced entity. `InboundEmailConsumer` subscribes to that entity's events and starts one `ResponderWorkflow` per email. The workflow drives `TriageAgent`, `ResponderAgent`, and `EmailEntity`; `EmailEntity` events project into `EmailsView`, which feeds the live UI and the `StuckReviewMonitor`. See the `flowchart` in [`../PLAN.md`](../PLAN.md#component-graph).

## Interaction sequence

The primary journey: an email is received, sanitized, triaged, drafted, then either auto-sent or held for a human. The `sequenceDiagram` in [`../PLAN.md`](../PLAN.md#interaction-sequence) shows the pause for sensitive replies.

## State machine

`EmailStatus` moves `RECEIVED → SANITIZED → DRAFTED`, then branches: routine replies go straight to `SENT`; sensitive replies go to `AWAITING_REVIEW`, then `APPROVED → SENT`, `REJECTED`, or `ESCALATED`. The `stateDiagram-v2` in [`../PLAN.md`](../PLAN.md#state-machine) is canonical; the generated UI must apply the state-label and edge-label CSS overrides so the labels render white-on-dark and unclipped.

## Entity model

`InboxQueue` enqueues emails; each `EmailEntity` emits its lifecycle events and projects into `EmailsView`. The `erDiagram` in [`../PLAN.md`](../PLAN.md#entity-model) shows the projection.
