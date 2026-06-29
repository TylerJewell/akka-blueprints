# Akka Sample: MCP GitHub Bot

A single AI agent that accepts natural-language requests and fulfills them by invoking a GitHub MCP server — listing repositories, reading issues, creating issues, and adding comments. Every write operation passes through a `before-tool-call` guardrail that checks intent before any state-mutating tool is dispatched, and an operator-controlled kill switch can halt all write access without redeployment.

Demonstrates the **single-agent** coordination pattern wired with two governance controls: a pre-tool guardrail that blocks destructive or mismatched tool calls before they reach GitHub, and a halt mechanism that lets an operator disable write operations at runtime.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A GitHub credential with `repo` scope (for full read+write) or `repo:read` (for read-only mode). Supply it the same way as the model-provider key: an existing shell env var (`GITHUB_TOKEN`), an env file, a secrets-store URI, or a one-time prompt. Akka never writes the token value to disk.

## Generate the system

```sh
cp -r ./single-agent.dev-code.mcp-github-bot  ~/my-projects/mcp-github-bot
cd ~/my-projects/mcp-github-bot
```

(Optional) Edit `SPEC.md` to point the `GitHubBotAgent` at a specific GitHub org or repository scope, or narrow the allowed tool list in the `before-tool-call` guardrail.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GitHubBotAgent** — an AutonomousAgent that receives a natural-language request, selects GitHub MCP tools, and returns a `BotResponse` carrying the outcome and the tool calls it made.
- **BotSessionWorkflow** — one workflow per session; runs a single agent task and records the outcome on the entity.
- **BotSessionEntity** — an EventSourcedEntity tracking the per-session lifecycle: submitted → running → completed / failed.
- **HaltFlagEntity** — a singleton EventSourcedEntity whose state is a boolean `writeHalted` flag; the guardrail reads it on every write-class tool call.
- **ToolCallGuardrail** — a `before-tool-call` hook that validates the pending tool call against the current halt flag and an allowlist of permitted tools. Blocked calls return a structured rejection so the agent can explain the denial to the user.
- **BotSessionView + BotSessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the GitHub owner/repo scope the agent is given in its system prompt; restrict or expand the tool allowlist.
- `SPEC.md §5` — extend `BotResponse` with fields relevant to your workflow (e.g., `linkedPR`, `assignedLabel`, `createdIssueNumber`).
- `prompts/github-bot.md` — narrow the agent's permitted operations (read-only, issues-only, a specific repository only).
- `eval-matrix.yaml` — swap the halt implementation for a production secret store rather than the in-process `HaltFlagEntity`.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a natural-language request → the agent invokes the right GitHub MCP tools → the response and tool-call log appear in the UI.
2. A user submits a write request while the halt flag is set → the guardrail blocks the tool call → the agent returns a denial message → no GitHub mutation occurs.
3. An operator toggles the halt flag off → the same user's next write request succeeds.
4. A request for a non-existent or disallowed tool → the guardrail blocks it → the agent's response explains the limitation.

## License

Apache 2.0.
