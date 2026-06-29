# Akka Sample: OAuth Tool Agent

A single agent receives a natural-language request, inspects the OAuth scopes attached to the caller's token, and decides whether the requested tool call is permitted before executing it. The scope check runs as a `before-tool-call` guardrail; the agent never issues the downstream API call if the token lacks the required scope.

Demonstrates the **single-agent** coordination pattern with one governance mechanism: a scope-validating guardrail that fires on every outbound tool call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the OAuth token store and downstream API calls are simulated in-process.

## Generate the system

```sh
cp -r ./single-agent.general.oauth-tool-agent  ~/my-projects/oauth-tool-agent
cd ~/my-projects/oauth-tool-agent
```

(Optional) Edit `SPEC.md` to point at a different set of OAuth scopes and tool definitions, or swap the seeded token fixtures for your own.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ToolCallerAgent** — an AutonomousAgent that receives a natural-language request plus a caller token id, resolves the token's scopes, and executes tool calls only where the scope check passes.
- **SessionWorkflow** — orchestrates token-resolve → agent-run → result-record per invocation.
- **SessionEntity** — an EventSourcedEntity holding the per-session lifecycle.
- **OAuthScopeGuardrail** — the `before-tool-call` guardrail that validates each proposed tool call against the resolved scope set before execution.
- **SessionView + SessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded token fixtures (token ids → scope sets) for your own OAuth provider's scope vocabulary.
- `SPEC.md §5` — extend `ToolCallRecord` with additional fields (e.g., `correlationId`, `clientApp`, `rateLimitBucket`).
- `prompts/tool-caller.md` — narrow the agent's available tools to the subset your deployment exposes.
- `eval-matrix.yaml` — add regulation anchors for the OAuth scopes your jurisdiction requires (e.g., data-access scopes under a privacy regulation).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a request with a token that holds the required scope → the agent executes the tool → the result appears in the UI.
2. A user submits a request with a token that lacks the required scope → the `before-tool-call` guardrail blocks the tool call → the agent returns a denial explanation instead.
3. A request whose token has partial scope coverage → the agent executes only the permitted tool calls and explains the blocked ones.
4. An expired token is submitted → the token-resolve step fails → the session transitions to `FAILED` before the agent is invoked.

## License

Apache 2.0.
