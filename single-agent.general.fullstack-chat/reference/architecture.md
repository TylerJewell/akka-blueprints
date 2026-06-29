# Architecture — gemini-fullstack

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ChatEndpoint` accepts a message submission, writes a `UserMessageAdded` event onto `ConversationEntity`, and immediately starts a `ConversationWorkflow` instance for the turn. The workflow's `generateStep` calls `ConversationContextBuilder` — a pure utility with no LLM dependency — to serialise the last N messages into a JSON attachment. It then calls `ChatAgent` — the single AutonomousAgent — with the context attachment and a short instruction string. Once a reply returns, the workflow writes `AgentReplyRecorded` back to the entity via `recordReply`. `ConversationView` projects every entity event into a read-model row; `ChatEndpoint` serves the read model over REST and SSE.

The graph has no second agent and no consumer. The context-building step is synchronous pure logic, not a separate Akka component. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two moments where latency concentrates:

1. The `generateStep` LLM call — bounded by the 60 s step timeout. The timeout gives room for model warmup and peak-load conditions without stalling the entity indefinitely.
2. The `recordStep` entity write — bounded by 10 s; this is almost always sub-millisecond for an in-process event-sourced entity.

The SSE delivery to the browser happens outside the workflow: as soon as `AgentReplyRecorded` lands on the entity, the view update propagates and the open SSE stream pushes the new row. The UI transitions from `AWAITING_REPLY` to `REPLY_RECORDED` without polling.

## State machine

Five states. The interesting paths:

- The happy path per turn is `ACTIVE → AWAITING_REPLY → REPLY_RECORDED → ACTIVE` — the conversation cycles back to `ACTIVE` whenever a new user message arrives after a reply lands.
- The initial state after creation is `CREATED`. The first `UserMessageAdded` moves it to `ACTIVE`.
- The `FAILED` terminal state is reached only when the workflow exhausts its retry budget. A `FAILED` conversation's event log is preserved — prior turns are still readable; only the failed turn has no reply.

## Entity model

`ConversationEntity` is the source of truth. It emits five event types. `ConversationView` projects every event into a row used by the UI. `ConversationWorkflow` both reads (`getConversation`) and writes (`markGenerating`, `recordReply`, `fail`) on the entity. The relationship between `ChatAgent` and `AgentReply` is "returns" — the agent's task result is the reply record that the workflow persists.

## Fullstack wiring

The embedded UI at `static-resources/index.html` is served by `AppEndpoint` directly from the Akka classpath. There is no reverse proxy, no CDN, and no separate server process. The frontend makes plain `fetch()` calls to `ChatEndpoint` for REST requests and opens an `EventSource` to the SSE endpoint for live updates. This makes the sample runnable with a single `java -jar` command and zero external dependencies beyond the JVM.
