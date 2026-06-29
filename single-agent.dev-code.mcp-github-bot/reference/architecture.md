# Architecture — mcp-github-bot

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one MCP-connected agent. `BotSessionEndpoint` accepts a natural-language request and writes a `SessionSubmitted` event onto `BotSessionEntity`. The same endpoint starts a `BotSessionWorkflow` instance, passing the GitHub token as a workflow-start parameter — the token is never stored in the entity. The workflow's `runAgentStep` calls `GitHubBotAgent` — the single AutonomousAgent — with the user request as `TaskDef.instructions(...)`.

Before any GitHub MCP tool call leaves the agent's loop, `ToolCallGuardrail` intercepts it. The guardrail checks two conditions: the tool name must be in the permitted set, and write-class tools must only proceed if `HaltFlagEntity` reports `writeHalted == false`. `HaltFlagEntity` is a standalone singleton EventSourcedEntity — it holds no LLM state, just a boolean flag that any operator can flip via `POST /api/halt/enable` or `POST /api/halt/disable`.

Once `GitHubBotAgent` returns a `BotResponse`, the workflow writes `SessionCompleted` to `BotSessionEntity`. `BotSessionView` projects entity events into the read model that the UI polls over SSE.

The graph has exactly one AutonomousAgent. `HaltFlagEntity` and `ToolCallGuardrail` are governance infrastructure — neither makes an LLM call.

## Interaction sequence

The sequence traces the happy path (J1): a read-only `list_issues` call with the halt flag unset. Two moments determine latency:

1. The agent's internal iteration: selecting the right tool, calling the MCP server, waiting for the GitHub response, then composing `BotResponse`. Bounded by the `runAgentStep` 90 s timeout.
2. `ToolCallGuardrail`'s synchronous read of `HaltFlagEntity` on every write-class dispatch. This is a local in-process componentClient call — sub-millisecond in normal operation.

For a multi-tool request (e.g., "list issues then create a summary issue"), the agent loops through multiple tool calls; each write-class call re-reads the halt flag. If the flag is toggled mid-session, the very next write-class call respects it.

## State machine

Four states on `BotSessionEntity`. The interesting path:

- The happy path is `SUBMITTED → RUNNING → COMPLETED`.
- `FAILED` catches both agent errors (MCP server unreachable, malformed response) and sessions where all write calls were blocked and the agent could not fulfill the request at all.
- There is no `HALTED` terminal state — a halt does not abort an in-flight session; it blocks individual tool calls within it. The session reaches `COMPLETED` even if every write call was blocked, because the agent still returns a `BotResponse` explaining the denials.

`HaltFlagEntity` has no lifecycle states — it is always active; its boolean value changes in place.

## Entity model

`BotSessionEntity` is the source of truth for session lifecycle and the agent's response. `HaltFlagEntity` is a separate entity with no lifecycle dependency on sessions — it can be toggled independently. `ToolCallGuardrail` reads `HaltFlagEntity` but does not write to it. `BotSessionWorkflow` reads and writes `BotSessionEntity`. `GitHubBotAgent` returns a `BotResponse` to the workflow; it has no direct entity access.

## Governance flow

For any tool call that exits the agent and reaches GitHub:

1. **Allowlist check** — the tool name is in the five-item permitted set. A hallucinated tool name never reaches the MCP server.
2. **Halt flag check** (write-class tools only) — `writeHalted == false` at the moment of dispatch. A flag flip by the operator takes effect within one tool call.
3. **GitHub MCP Server** — the actual network call, made only after both checks pass.

Removing either check opens an explicit gap. The allowlist alone does not prevent write operations during an incident; the halt alone does not prevent a hallucinated tool from attempting to call a non-existent endpoint.
