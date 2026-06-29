# Architecture — multi-model-chatbot

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ChatEndpoint` accepts a message submission and writes a `MessageReceived` event onto `ConversationEntity`. The `MessageSanitizer` Consumer subscribes, redacts PII from the user text, and writes the sanitized text back via `attachSanitized`. The same Consumer then starts a `ChatWorkflow` instance for the turn. The workflow's `replyStep` calls `ChatAgent` — the single AutonomousAgent — with the full conversation history plus the sanitized message as inline task instructions. The agent's `before-agent-response` guardrail (`ReplyGuardrail`) validates each candidate reply via `ContentPolicyChecker` before it exits the agent loop. Once a reply passes, the workflow writes `ReplyGenerated` to the entity. `ConversationView` projects every entity event into a read-model row; `ChatEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The content policy checker (`ContentPolicyChecker`) is a deterministic rule-based component — not an LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct phases of processing occur before the reply reaches the user:

1. The `MessageSanitizer` subscription lag between `MessageReceived` and `MessageSanitized` — sub-second in normal operation, as the Consumer runs in-process with the entity.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `ConversationEntity` every 1 s up to its 15 s timeout, advancing as soon as the target turn's status equals `SANITIZED`.

The agent call itself is bounded by `replyStep`'s 60 s timeout. `ReplyGuardrail` runs synchronously inside each candidate-response hook call — no additional latency beyond the in-process check.

## State machine

Each message turn inside a session cycles through four states. The interesting paths:

- The happy path is `RECEIVED → SANITIZED → REPLIED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `RECEIVED`, and an agent error (including guardrail-iteration exhaustion) during `SANITIZED`. A `FAILED` turn is visible in the chat thread — the UI shows the partial state so the user knows the reply did not arrive.
- There is no session-level terminal "CLOSED" state forced by the system. Sessions close on an explicit user action or a deployer-configured inactivity timeout.

## Entity model

`ConversationEntity` is the source of truth for every session. It emits seven event types. `ConversationView` projects every event into a row used by the UI. `MessageSanitizer` subscribes to entity events to compute the sanitized text. `ChatWorkflow` both reads (`getSession`) and writes (`attachSanitized`, `recordReply`, `failTurn`) on the entity. The relationship between `ChatAgent` and `ChatReply` is "returns" — the agent's task result is the reply record stored on the turn.

## Defence-in-depth governance flow

For any reply that reaches the user, the message passed through:

1. **PII sanitizer** — the model never sees real email addresses, phone numbers, or account identifiers; the audit log retains the raw form.
2. **ChatAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — harmful content, off-topic material, and PII re-exposure are caught before the response leaves the agent loop.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover. A deployer adding a third check (e.g., a topic-relevance classifier) would wire it as an additional policy rule inside `ReplyGuardrail`, not as a separate agent.
