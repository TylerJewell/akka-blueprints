# Akka Sample: Async Agent Endpoint

A single `CodeAgent` is served behind a Starlette ASGI endpoint. Incoming requests are dispatched with `anyio.to_thread.run_sync` so the event loop remains responsive while the agent runs. Before the agent executes any tool call, a sandboxing guardrail checks that generated code does not escape the allowed execution context.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that intercepts every tool invocation the agent emits, inspects the generated code, and rejects calls that violate the sandbox policy.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the code-execution sandbox is in-process and the agent's tool calls are simulated in mock mode.

## Generate the system

```sh
cp -r ./single-agent.general.async-agent-endpoint  ~/my-projects/async-agent-endpoint
cd ~/my-projects/async-agent-endpoint
```

(Optional) Edit `SPEC.md` to narrow the agent's allowed tool set (e.g., restrict to read-only file operations or a specific list of safe modules).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CodeRunnerAgent** — an AutonomousAgent that accepts a natural-language task, generates Python code to satisfy it, and returns a typed `RunResult` containing the output or error.
- **AgentRunWorkflow** — orchestrates submit → run → record per request, with a `before-tool-call` guardrail wired into the agent loop.
- **AgentRunEntity** — an EventSourcedEntity holding the per-run lifecycle.
- **RunView + RunEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add seeded task prompts that exercise different tool categories (file I/O, HTTP, data transformation).
- `SPEC.md §5` — extend `RunResult` with fields for timing, memory usage, or intermediate step traces.
- `prompts/code-runner.md` — tighten the allowed module list or add a domain-specific preamble (e.g., restrict to pandas + numpy for a data-science deployer).
- `eval-matrix.yaml` — wire a real sandboxing back-end (e.g., a container-isolated executor) by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task prompt → the agent runs → the result appears in the UI within the step timeout.
2. The agent emits a tool call that imports a forbidden module → the `before-tool-call` guardrail rejects it → the agent retries with a compliant call → the UI never shows the rejected tool invocation.
3. A run whose generated code raises an uncaught exception is recorded as `FAILED` with the exception message preserved.
4. Concurrent requests are handled without event-loop blocking — a slow run does not delay the response to other in-flight requests.

## License

Apache 2.0.
