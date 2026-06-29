# Akka Sample: Function Calling Agent

A single agent receives a user query, decides which tools to call, invokes them in a loop, and returns a final answer once it has enough information. The tool-call loop is governed by two guardrails: one that validates each tool invocation before it fires, and one that filters the final answer before it reaches the caller.

Demonstrates the **single-agent** coordination pattern in the general domain. One `FunctionCallingAgent` (AutonomousAgent) drives the entire loop; the surrounding components handle request lifecycle, state persistence, and read-model projection.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the tool registry lives in-process and all tool implementations are simulated.

## Generate the system

```sh
cp -r ./single-agent.general.function-calling-agent-baseline  ~/my-projects/function-calling-agent
cd ~/my-projects/function-calling-agent
```

(Optional) Edit `SPEC.md` to replace the seeded in-process tools with real external calls — e.g., swap the weather tool stub for a live weather API or add a database-query tool.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FunctionCallingAgent** — an AutonomousAgent that receives a query plus a tool registry, calls tools as needed, and returns a typed `AgentAnswer`.
- **AgentRunWorkflow** — orchestrates submit → await-start → run → score per submitted query.
- **AgentRunEntity** — an EventSourcedEntity holding the per-run lifecycle.
- **ToolDispatcher** — a Consumer that subscribes to `ToolCallRequested` events, executes the named tool, and emits `ToolCallCompleted` back to the entity.
- **AgentRunView + AgentRunEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap or extend the seeded tool set (calculator, dictionary, date-formatter) for domain-specific tools.
- `SPEC.md §5` — add fields to `AgentAnswer` (e.g., `List<ToolCallRecord>` as a trace, `confidence: double`).
- `prompts/function-calling-agent.md` — narrow the agent's persona and tool-use policy for a specific domain (customer support, data analytics, etc.).
- `eval-matrix.yaml` — replace the in-process `ToolCallValidator` with a real policy engine if tool calls touch external systems.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a query → the agent invokes tools → a final answer appears in the UI.
2. A tool call with an invalid argument is blocked by the `before-tool-call` guardrail; the agent receives a structured rejection and tries a corrected call.
3. The agent's first draft answer on a flagged query is blocked by the `before-agent-response` guardrail; the agent revises and a clean answer is returned.
4. Every recorded run has a tool-call trace visible on the same UI card.

## License

Apache 2.0.
