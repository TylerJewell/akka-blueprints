# Architecture — akka-stateful-memory-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ConversationEndpoint` accepts a user message, writes `UserMessageReceived` onto `ConversationEntity`, and starts a `ConversationWorkflow` instance. The workflow's `callAgentStep` invokes `MemoryAgent` — the single AutonomousAgent — with the current `persona` block, `human` block, and recent history formatted as task instructions. The agent returns an `AgentTurnResult` containing the reply text and any proposed memory patches. The workflow writes `AgentResponseReady` back to the entity. The `MemoryWriterConsumer` Consumer picks up the event, sanitizes PII from each proposed patch, and calls `ConversationEntity.applyMemoryPatch` for each one. On every tenth turn, the workflow also runs `DriftEvaluator` — a deterministic rule-based scorer — to check whether the human block has drifted toward demographic stereotyping. `ConversationView` projects every entity event into a row the UI reads over REST and SSE.

There is no second agent. `DriftEvaluator` is a deterministic rule-based scorer — not an LLM. That is what makes this a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces a single turn on the happy path (J1). Two notable moments:

1. The `MemoryWriterConsumer` subscription lag between `AgentResponseReady` and `MemoryPatchApplied` — sub-second in normal operation, because the sanitizer pipeline is in-process.
2. The `evalStep` fires only on every tenth turn. On other turns the step definition exists but the workflow skips it, so `latestDrift` on the entity state retains its previous value until the next eval cycle.

The agent call itself is bounded by `callAgentStep`'s 60 s timeout. The sanitizer in `applyMemoryStep` is bounded by 15 s. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

The conversation entity stays in `ACTIVE` for its entire useful life, cycling through repeated `UserMessageReceived → AgentResponseReady → MemoryPatchApplied` event sequences. `DriftEvaluated` is an annotation event; it does not change the status. The only terminal transition is `ARCHIVED`. There is no `FAILED` status for the conversation as a whole — a failed turn is recorded on the workflow; the conversation itself remains `ACTIVE` so the user can send another message.

## Entity model

`ConversationEntity` is the source of truth. It emits six event types. `ConversationView` projects every event into a row used by the UI. `MemoryWriterConsumer` subscribes to entity events to apply sanitized memory patches. `ConversationWorkflow` both reads (`getConversation`) and writes (`recordAgentResponse`, `applyMemoryPatch`, `recordDrift`) on the entity. `MemoryAgent`'s relationship to `AgentTurnResult` is "returns" — the agent's task result carries both the reply and the proposed patches.

## Defence-in-depth governance flow

For any memory patch that lands in the entity log, the content passed through:

1. **PII sanitizer** — `MemoryWriterConsumer` strips identifiers before persisting; the model never reads raw PII from prior turns because it was never written to the block.
2. **MemoryAgent** — one model call per turn, one structured output carrying the reply and proposed patches.
3. **Periodic drift evaluator** — every ten turns, the human block is scored for demographic-stereotype collapse; a flagged session is surfaced to a human reviewer through the UI.

Each step is independent. The sanitizer runs unconditionally; the drift check runs periodically. Neither silently compensates for the other.
