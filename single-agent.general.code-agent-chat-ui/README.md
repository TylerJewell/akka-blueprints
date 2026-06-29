# Akka Sample: Gradio UI Code Agent

A single code-executing agent accepts natural-language requests in a streaming chat UI, runs web searches and code execution to answer them, and re-plans every 3 steps. Two guardrails sit around it: one validates every tool call before it executes (code execution requires a sandbox boundary), and one validates the agent's final message before it reaches the user.

Demonstrates the **single-agent** coordination pattern in the general domain. One `CodeAssistantAgent` (AutonomousAgent) drives all reasoning; the surrounding components govern its tool calls and outbound messages.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — web search and code execution are simulated in-process; no external sandbox required.

## Generate the system

```sh
cp -r ./single-agent.general.code-agent-chat-ui  ~/my-projects/code-agent-chat-ui
cd ~/my-projects/code-agent-chat-ui
```

(Optional) Edit `SPEC.md` to change the planning interval (currently every 3 steps), extend the allowed tool set, or tighten the output guardrail's content policy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CodeAssistantAgent** — an AutonomousAgent that processes chat turns, calls `WebSearchTool` and `CodeExecutionTool`, replans every 3 steps, and streams a response.
- **ChatSessionEntity** — an EventSourcedEntity holding the per-session message history and tool-call log.
- **ChatSessionWorkflow** — orchestrates session initialization → active turn processing → idle.
- **ToolCallValidator** — a Consumer that runs the `before-tool-call` guardrail, approving or blocking each pending tool invocation.
- **ResponseGuardrail** — wired on the `before-agent-response` hook; validates the agent's output before it reaches the chat stream.
- **ChatView + ChatEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the planning interval or seed conversation starters shown on the App UI tab.
- `SPEC.md §5` — add fields to `ChatMessage` (e.g., `citations`, `confidenceScore`) or extend `ToolCallRecord` with sandbox metadata.
- `prompts/code-assistant.md` — restrict the agent to a specific domain (data analysis, infrastructure queries, etc.).
- `eval-matrix.yaml` — swap the simulated sandbox in `G2` for a real container-based executor by naming it in the implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user sends a natural-language question → the agent searches, writes code, and returns a streamed answer visible in the chat UI.
2. The agent attempts a disallowed tool call → the `before-tool-call` guardrail blocks it → the agent receives the rejection and continues without the blocked call.
3. The agent produces a response containing a policy-violating pattern → the `before-agent-response` guardrail rejects it → the agent retries and produces a clean response.
4. Every tool invocation is recorded in the session's `toolCallLog`, visible in the session detail panel.

## License

Apache 2.0.
