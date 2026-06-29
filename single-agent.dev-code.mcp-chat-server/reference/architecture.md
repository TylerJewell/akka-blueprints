# Architecture — mcp-chat-server

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ChatEndpoint` accepts a new message, writes a `MessageReceived` event to `ConversationEntity`, and starts a `ChatWorkflow` instance keyed to that turn. The workflow's `runAgentStep` calls `ChatAgent` — the single AutonomousAgent — with the user's message and the conversation history formatted as the task instructions. The agent calls zero or more MCP tools via the registered `MCPClient`; each tool invocation passes through `ToolCallGuardrail` before any HTTP request leaves the JVM. Once the agent has the data it needs, it forms a candidate `ChatReply`; that candidate passes through `ReplyGuardrail` before it is returned to the workflow. The workflow then calls `ConversationEntity.recordReply` to append the turn. `ConversationView` projects every entity event into a read-model row; `ChatEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. Both guardrails are registered on the single `ChatAgent` — they are hooks in the agent loop, not separate LLM components. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). The key pause points:

1. The `runAgentStep` MCP round-trip — the agent issues a tool call, `MockMcpServer` (or a real MCP server) responds, and the agent incorporates the result before forming its reply.
2. The `before-tool-call` cut-point — `ToolCallGuardrail` runs synchronously inside the agent loop, between the agent's decision to call a tool and the actual HTTP request. Blocked here means the tool call never leaves the JVM.
3. The `before-agent-response` cut-point — `ReplyGuardrail` runs synchronously between the agent forming its reply and the reply being returned to the workflow. Rejected here means the workflow never sees the bad response.

The full turn is bounded by `runAgentStep`'s 60 s timeout, which accommodates both LLM latency and MCP round-trips.

## State machine

The turn lifecycle inside `ConversationEntity` has four states. The interesting paths:

- Happy path: `RECEIVED → RUNNING → REPLIED`.
- Failure path: `RUNNING → FAILED` when either the agent exhausts its iteration budget (all 4 iterations rejected by `ReplyGuardrail`) or the workflow step times out / exceeds its retry budget.
- There is no `PENDING_REVIEW` state — this blueprint routes replies directly to the user. A deployer who needs human-in-the-loop approval would insert an additional state between `REPLIED` and delivery.
- `ConversationEntity` as a whole has an `ACTIVE` / `CLOSED` status separate from the per-turn state machine. Closing a conversation prevents new `MessageReceived` events.

## Entity model

`ConversationEntity` is the source of truth. It emits six event types. `ConversationView` projects every event into a row with an embedded turn list used by the UI. `ChatWorkflow` both reads (`getConversation` via the view) and writes (`markRunning`, `recordReply`, `failTurn`) on the entity. The relationship between `ChatAgent` and `ChatReply` is "returns" — the agent's task result is the reply record, which is then stored on the entity by the workflow.

`ToolCall` audit records are embedded inside `ChatReply` and therefore inside the `ReplyRecorded` event. Every tool the agent attempted — allowed or blocked — is visible in the entity log.

## Governance flow

For any reply that reaches the user, the message passed through:

1. **ToolCallGuardrail (before-tool-call)** — every MCP tool invocation is allow-listed, URL-validated, and size-checked before it fires. Blocked calls cannot reach the external MCP server.
2. **ChatAgent** — one model call per iteration, structured output.
3. **ReplyGuardrail (before-agent-response)** — structural validity, length bounds, and content-pattern checks run before the reply leaves the agent loop.

Both cut-points are independent. Removing one opens an explicit gap the other does not silently cover. A deployer who adds a third check (e.g., a PII scan on tool outputs before they enter the agent's context) would add a third hook rather than modifying either existing guardrail.
