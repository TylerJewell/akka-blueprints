# Architecture — slack-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `MessageEndpoint` accepts an inbound message submission and writes a `MessageReceived` event onto `MessageEntity`. The `SecretSanitizer` Consumer subscribes, strips secrets from the message context, and writes the sanitized form back via `attachSanitized`. The same Consumer then starts a `MessageWorkflow` instance. The workflow's `replyStep` calls `ChannelAssistantAgent` — the single AutonomousAgent — with the channel routing context as `TaskDef.instructions(...)` and the sanitized message window as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ReplyGuardrail`) validates each candidate response. Once a reply passes, the workflow writes `ReplyRecorded` and runs `AuditRecorder` in `auditStep`. The audit record lands as `AuditCompleted`. `MessageView` projects every entity event into a read-model row; `MessageEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `AuditRecorder` is deterministic — it inspects task result metadata, not an LLM output. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system waits:

1. The `SecretSanitizer` subscription lag between `MessageReceived` and `MessageSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `MessageEntity` every 1 s up to its 15 s timeout, advancing as soon as `message.sanitized().isPresent()` returns true.

The agent call itself is bounded by `replyStep`'s 60 s timeout. The `auditStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The important paths:

- The happy path is `RECEIVED → SANITIZED → REPLYING → REPLY_RECORDED → AUDITED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `RECEIVED`, and an agent error (or guardrail-exhaustion) during `REPLYING`. A `FAILED` message's prior data is preserved on the entity — the UI shows the partial state for the operator.
- There is no `POSTED` state. Whether the reply actually reaches the Slack API is a deployment concern handled outside the agent loop. The blueprint stops at `AUDITED`.

## Entity model

`MessageEntity` is the source of truth. It emits six event types. `MessageView` projects every event into a row used by the UI. `SecretSanitizer` subscribes to entity events to compute the sanitized form. `MessageWorkflow` both reads (`getMessage`) and writes (`markReplying`, `recordReply`, `recordAudit`, `fail`) on the entity. The relationship between `ChannelAssistantAgent` and `AssistantReply` is "returns" — the agent's task result is the reply record.

## Defence-in-depth governance flow

For any reply that lands in the entity log, the message context passed through:

1. **Secret sanitizer** — the model never sees credentials; the audit log retains the raw trigger text.
2. **ChannelAssistantAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — bad parses, invalid action types, and any secret-pattern strings in the response text are caught before the reply leaves the agent loop.
4. **AuditRecorder** — every well-formed reply gets a candidate/rejected count record so operators can see at a glance how many guardrail retries each message triggered.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
