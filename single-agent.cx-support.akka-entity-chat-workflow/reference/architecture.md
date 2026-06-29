# Architecture — entity-workflow-chat

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call per turn. `ChatEndpoint` accepts a new conversation or an incoming message and writes events onto `ConversationEntity`. When a `MessageReceived` event lands, the `MessageSanitizer` Consumer subscribes, strips PII, and writes the sanitized form back via `attachSanitized`. The same Consumer signals `ChatWorkflow` to resume from `awaitTurnStep`. The workflow assembles a context package from the entity's turn log and compaction summaries, then calls `ConversationAgent` — the single AutonomousAgent — with the context as a `TaskDef.attachment("context.json", ...)`. Once a reply arrives, the workflow writes `AgentReplied` and `completeTurn` onto the entity, estimates the accumulated token count, and either loops back to `awaitTurnStep` or transitions to `compactStep`. In `compactStep`, `ContextCompactor` produces a `ConversationSummary` (no LLM call — fully deterministic), which is written onto the entity before the workflow loops back to `awaitTurnStep`. `ConversationView` projects every entity event into a read-model row; `ChatEndpoint` serves the read model over REST and SSE.

The graph has no second agent. History compaction is handled by `ContextCompactor`, a deterministic in-process class. That is what makes this a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path for a single user turn (J1). Note two structural pause points:

1. The `MessageSanitizer` subscription lag between `MessageReceived` and `MessageSanitized` — sub-second in normal operation.
2. The `awaitTurnStep` genuine pause inside the workflow — the workflow is idle, consuming no resources, until a new user message arrives and resumes it. There is no polling or timer.

The agent call itself is bounded by `agentStep`'s 60 s timeout. The `recordStep` and optional `compactStep` are in-process and finish in milliseconds.

## State machine

Six states. The significant paths:

- The conversation lifecycle is `OPEN → AGENT_THINKING → OPEN (repeat) → CLOSED`.
- When the token budget is exceeded the entity enters `COMPACTING` briefly before returning to `OPEN`; from the customer's perspective the conversation continues uninterrupted.
- `FAILED` is reachable from `AGENT_THINKING` if the agent's step exhausts its retry budget. The entity retains all prior turns so a support agent can review the partial conversation.
- `CLOSED` is terminal; there is no re-open path. A new conversation must be started for follow-up questions.

## Entity model

`ConversationEntity` is the source of truth. It emits seven event types. `ConversationView` projects every event into a read-model row. `MessageSanitizer` subscribes to entity events to compute the sanitized form and resume the workflow. `ChatWorkflow` both reads (`getConversation`) and writes (`recordReply`, `completeTurn`, `recordCompaction`, `fail`) on the entity. `ConversationAgent`'s task result is the `AgentReply` record; the workflow writes this onto the entity as an `AgentReplied` event.

## Human-in-the-loop and PII governance flow

For every agent call that fires, the user message has passed through:

1. **User POST** — the customer's explicit action. The workflow is literally paused until this arrives; there is no autonomous trigger.
2. **PII sanitizer** — the model never sees customer identifiers; the audit log on the entity retains raw text.
3. **ConversationAgent** — one model call, one structured reply per turn.
4. **Context compaction** — deterministic rule-based summariser keeps the context window bounded without a second LLM call.

Each step is independent. The human-in-the-loop property is structural: removing `awaitTurnStep`'s pause would require rewriting the workflow, not merely toggling a flag.
