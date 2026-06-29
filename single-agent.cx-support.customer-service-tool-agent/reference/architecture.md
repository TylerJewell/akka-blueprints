# Architecture — customer-service-tool-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SupportEndpoint` accepts a new conversation (or follow-up message) and writes the corresponding events onto `ConversationEntity`. It then starts a `ConversationWorkflow` instance. The workflow's `agentStep` calls `SupportAgent` — the single AutonomousAgent — with the formatted conversation history as the task instructions. Two guardrails are wired on the agent: `ToolCallGuardrail` intercepts every tool call before it executes, and `ReplyGuardrail` intercepts every candidate reply before it leaves the agent loop. Once a reply passes both checks, the workflow calls `ConversationEntity.recordReply`. The `ReplyScreener` Consumer then subscribes to the `AgentReplied` event, runs a PII redaction pass over the reply text, and writes the screened form back via `recordScreened`. `ConversationView` projects every entity event into a read-model row; `SupportEndpoint` serves the read model over REST and SSE.

The graph has no second agent. The PII screener (`ReplyScreener`) is regex-based, and the tool-call gate (`ToolCallGuardrail`) is a deterministic policy check. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct control boundaries:

1. `ToolCallGuardrail` fires synchronously inside the agent loop before any write tool executes — the tool call either proceeds or returns a synthetic block result the agent can read.
2. `ReplyGuardrail` fires synchronously inside the agent loop before the candidate reply is returned — a failing reply triggers a retry within the iteration budget.
3. `ReplyScreener` runs asynchronously in a Consumer after the reply lands on the entity — `screenStep` polls up to 15 s for `ReplyScreened`, keeping the log hygiene path out of the agent's critical path.

The agent call itself is bounded by `agentStep`'s 90 s timeout, which accommodates LLM latency plus multiple tool-call round trips in the same task.

## State machine

Six states. The key paths:

- Happy path: `OPEN → ACTIVE → REPLIED`. A follow-up message cycles back to `ACTIVE` for the next turn.
- Escalation path: `ACTIVE → ESCALATED` when `ToolCallGuardrail` blocks a write call and the agent opens an escalation ticket. `ESCALATED` is terminal for the automated system; a human operator continues the session.
- Resolution path: `REPLIED → RESOLVED` when the endpoint receives an explicit resolve command (e.g., the customer confirms their issue is fixed).
- Failure paths: `OPEN → FAILED` on workflow startup error; `ACTIVE → FAILED` on agent error or guardrail-exhaustion after 4 iterations.

A `FAILED` conversation preserves the partial event log — the UI shows all turns up to the failure point.

## Entity model

`ConversationEntity` is the source of truth. It emits eight event types. `ConversationView` projects every event into a read-model row used by the UI. `ReplyScreener` subscribes to entity events to compute the screened form. `ConversationWorkflow` both reads (`getConversation`) and writes (`markActive`, `recordReply`, `recordScreened`, `escalate`, `fail`) on the entity. `SupportAgent` returns `AgentReply` — the agent's task result is the reply record.

## Defence-in-depth governance flow

For any reply that reaches the UI, the session passed through:

1. **ToolCallGuardrail** — every write tool call was within policy before it executed; blocked calls left no data side-effects.
2. **SupportAgent** — one model call per turn, producing a grounded tool-use result.
3. **ReplyGuardrail** — no raw PAN, SSN, or ungrounded claim in the reply text.
4. **ReplyScreener** — the log record carries only redacted PII; the raw form is retained on the entity for audit.

Each layer is independent. A gap in one does not silently transfer responsibility to another.
