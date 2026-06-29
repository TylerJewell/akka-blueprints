# Akka Sample: Chat Server (MCP)

A Starlette-based web chat server that fronts a single CodeAgent augmented with external tools supplied via an MCP client. Users send natural-language messages; the agent calls whichever MCP tools are needed, then streams a response back to the browser. Graceful shutdown drains in-flight requests before the JVM exits.

Demonstrates the **single-agent** coordination pattern in the developer-tooling domain. Two governance mechanisms surround the agent: a `before-tool-call` guardrail that validates every MCP tool invocation before it reaches an external service, and a `before-agent-response` guardrail that validates the agent's final reply before it is streamed to the user.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The MCP client connects to registered tool servers at startup; a bundled mock MCP server handles the out-of-the-box path.

## Generate the system

```sh
cp -r ./single-agent.dev-code.mcp-chat-server  ~/my-projects/mcp-chat-server
cd ~/my-projects/mcp-chat-server
```

(Optional) Edit `SPEC.md` to register a different MCP server URL (e.g., point `ChatServerConfig.mcpServerUrl` at a real filesystem or code-execution MCP server rather than the bundled mock).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ChatAgent** — an AutonomousAgent that accepts a user message as its task, calls MCP tools as needed via the registered `MCPClient`, and returns a `ChatReply`.
- **ChatWorkflow** — orchestrates message-received → agent-run → reply-recorded per submitted message.
- **ConversationEntity** — an EventSourcedEntity holding per-conversation history and status.
- **ToolCallGuardrail** — the `before-tool-call` hook; validates every MCP tool invocation before it fires.
- **ReplyGuardrail** — the `before-agent-response` hook; validates the agent's reply before it is sent to the user.
- **ConversationView + ChatEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded MCP server registration for your own (e.g., point `mcp.server-url` at a filesystem, code-execution, or search MCP server in `application.conf`).
- `SPEC.md §5` — extend `ChatReply` with domain-specific fields (e.g., `toolsUsed`, `confidenceLabel`, `citedSources`).
- `prompts/chat-agent.md` — narrow the agent's role (a developer-tools deployer might restrict it to code-related questions only; a DevOps deployer might restrict it to infrastructure queries).
- `eval-matrix.yaml` — wire a real content-filter library under the `before-agent-response` guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user sends a message → the agent calls one or more MCP tools → a reply streams back to the chat UI within 30 s.
2. The agent attempts to invoke a blocked MCP tool → the `before-tool-call` guardrail rejects the call → the agent recovers and replies without using the blocked tool.
3. The agent produces a reply that fails the content check → the `before-agent-response` guardrail rejects it → the agent retries → a valid reply lands.
4. Two simultaneous conversations remain isolated; each `ConversationEntity` holds its own turn history and status.

## License

Apache 2.0.
