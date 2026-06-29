# Architecture — react-chatbot

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ChatEndpoint` accepts a new message and writes a `TurnStarted` event onto `ConversationEntity`. The same endpoint then starts a `ConversationWorkflow` keyed to that turn. The workflow's `agentStep` calls `ChatAgent` — the single AutonomousAgent — with the formatted conversation history plus the user message as `TaskDef.instructions(...)`. The agent calls in-process tools via `ToolRegistry` (a synchronous stub dispatcher, not a network call), iterates until it has enough information, and produces a candidate `ChatReply`. That candidate passes through `ReplyGuardrail` — the `before-agent-response` hook — before leaving the agent loop. If the guardrail rejects it, the loop retries within its 3-iteration budget. Once a reply passes, the workflow's `recordStep` calls `ConversationEntity.recordReply(turnId, reply)`. `ConversationView` projects every entity event into a read-model row; `ChatEndpoint` serves the view over REST and SSE.

The graph has no second agent. `ToolRegistry` is in-process, deterministic, and LLM-free. `ReplyGuardrail` is a rule-based check, not a model call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note the tool-call exchange between the agent and `ToolRegistry`: the agent may call one or more tools before deciding it has enough information to produce a final reply. Each tool call is recorded in the `ChatReply.toolCallTrace` list so the UI can surface the reasoning chain.

The `before-agent-response` hook fires exactly once, on the candidate final answer — not on intermediate tool-call steps. This means the agent can call tools freely; only the outbound reply is gated.

## State machines

Two state machines are in play:

1. **Conversation-level** (`ConversationEntity`): a conversation starts in `ACTIVE` on `ConversationStarted` and moves to `CLOSED` on `ConversationClosed`. Turns are appended to the conversation's `turns` list as events arrive; the conversation-level status does not change per turn.
2. **Turn-level** (within each `Turn` record): `PROCESSING → COMPLETED` on `ReplyRecorded`; `PROCESSING → FAILED` on `TurnFailed` (guardrail exhaustion or unrecoverable agent error).

Keeping the two state machines separate means the conversation stays `ACTIVE` even when a single turn fails — the user can send another message without starting a new conversation.

## Entity model

`ConversationEntity` is the source of truth. It emits five event types. `ConversationView` projects every event into a row used by the UI list and detail pane. `ConversationWorkflow` both reads (`getConversation`) and writes (`recordReply`, `failTurn`) on the entity. The relationship between `ChatAgent` and `ChatReply` is "returns" — the agent's task result is the reply record. `ToolRegistry` is a dependency of `ChatAgent`, not of the entity or the workflow.

## Governance flow

For any reply that reaches the user, the message path passed through:

1. **ChatAgent** — one model call, zero or more tool calls, one structured output.
2. **before-agent-response guardrail** — system-prompt disclosure, PII fabrication, and scope-breach patterns are checked before the response leaves the agent loop. A bad reply never reaches `ConversationEntity`.

The guardrail check is independent of the tool-call loop. Removing it opens an explicit gap: the agent could produce a policy-violating reply on any iteration, and there would be no catch.
