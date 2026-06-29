# Akka Sample: Deferred Tool-Discovery Harness

A single discovery agent receives a user task, searches the tool catalog for relevant schemas on demand, passes discovered tools through an allowlist guardrail, and executes the task — returning a structured `TaskResult` with the output and the list of tools actually used.

Demonstrates the **single-agent** coordination pattern with one governance mechanism: a `before-tool-invocation` guardrail that verifies every discovered tool against a configured allowlist before the agent binds the schema. Tools whose ids do not appear in the allowlist are stripped before the agent can call them.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The tool catalog is seeded from an in-process JSONL file; no external registry is required to run the sample.

## Generate the system

```sh
cp -r ./single-agent.general.tool-search-discovery  ~/my-projects/tool-search-discovery
cd ~/my-projects/tool-search-discovery
```

(Optional) Edit `SPEC.md §3` to change the seeded tool catalog entries (under `src/main/resources/sample-events/tool-catalog.jsonl`) or adjust the allowlist in `src/main/resources/allowlist.json`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DiscoveryAgent** — an AutonomousAgent that receives the user query in the task definition and calls the `tool_search` tool to locate relevant schemas, then executes the task with only the discovered tools.
- **DiscoveryWorkflow** — orchestrates receive → discover → execute per submitted task.
- **TaskEntity** — an EventSourcedEntity holding the per-task lifecycle.
- **ToolCatalogConsumer** — a Consumer that subscribes to `ToolCatalogUpdated` events and keeps the in-process registry current.
- **TaskView + TaskEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A `before-tool-invocation` guardrail (`ToolAllowlistGuardrail`) that strips tools not in the allowlist before the agent can invoke them.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the seeded tool catalog with your own entries (a deployer adding real API-wrapper tools would list them in `tool-catalog.jsonl`).
- `SPEC.md §5` — extend `ToolEntry` with versioning fields (e.g., `schemaVersion`, `deprecatedAt`) for catalog lifecycle management.
- `prompts/discovery-agent.md` — narrow the agent's search behavior (a deployer in a regulated industry might constrain it to tools whose `category` field matches an approved set).
- `reference/allowlist.json` (generated) — add or remove tool ids to change what the guardrail permits.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task query → the agent discovers relevant tools → executes the task → the result appears in the UI with the tool list.
2. The agent attempts to invoke a tool not in the allowlist → the guardrail strips it before invocation → the task completes using only permitted tools.
3. A task query that matches zero catalog entries completes gracefully with an empty tool list and a result explaining no tools were applicable.
4. The tool catalog is updated at runtime → the agent's next task picks up the new entry without a service restart.

## License

Apache 2.0.
