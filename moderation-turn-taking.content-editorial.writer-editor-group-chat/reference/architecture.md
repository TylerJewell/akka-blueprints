# Architecture

The system is a moderated group chat. A durable Workflow acts as the group chat manager: it grants the floor to one agent at a time (RequestToSpeak round-robin), runs a guardrail on each agent's content before broadcast, and writes turns to an event-sourced conversation. A read model streams the transcript to the UI.

The four diagrams below are the canonical mermaid source. The generated `index.html` renders them on the Architecture tab with the Lesson 24 state-label CSS overrides and theme variables.

## Component graph

The flowchart in `PLAN.md` (Component graph) shows the runtime wiring. A topic enters through `ChatEndpoint` or `TopicSimulator` into `InboundTopicQueue`. `ConversationConsumer` starts one `GroupChatWorkflow` per queued topic. The workflow alternates `WriterAgent` and `EditorAgent`, writing each turn to `ConversationEntity`. `ConversationEntity` events fan out to `TurnView` (read model) and `TurnEvalConsumer` (event-triggered eval). `StallMonitor` escalates stalled conversations. `ChatEndpoint` and `AppEndpoint` serve the API and UI.

## Interaction sequence

The sequence in `PLAN.md` (Interaction sequence) traces one submission through the round-robin loop. Each round grants the floor to the Writer, runs the before-agent-response guardrail, publishes (or blocks) the Writer turn, then grants the floor to the Editor and publishes the Editor's decision turn. The loop ends on approval or at the round limit. `Note over` blocks mark the floor-grant and guardrail moments.

## State machine

The `stateDiagram-v2` in `PLAN.md` (State machine) is the `ConversationEntity` lifecycle: `IN_PROGRESS` on open, self-looping on each published or blocked turn, then a terminal transition to `APPROVED`, `MAX_ROUNDS_REACHED`, or `ESCALATED`, and finally `CLOSED`.

## Entity model

The `erDiagram` in `PLAN.md` (Entity model) shows `Conversation` containing ordered `Turn` rows, emitting conversation events, and projecting into the `TurnView` row. `Turn` carries the speaker, content, round, eval score, and blocked flag.

## Why these primitives

- The manager is a `Workflow` because the round-robin loop is durable and restartable — a crash mid-conversation resumes from the last completed step.
- The conversation is an `EventSourcedEntity` because the transcript is the source of truth and every turn is a stored event.
- The eval is a `Consumer` on conversation events so the Editor's decision is scored without coupling the score into the manager's critical path.
