# Akka Sample: Managed-Harness Tool-Use Agent

A single OperationsAgent answers natural-language questions by calling a declared set of tools — query_metrics, list_resources, fetch_logs, compute_stats — in whatever order and however many times the task requires. The managed runtime executes the tool-calling loop end-to-end; a `before-tool-invocation` guardrail checks every tool call against the allowlist before the runtime dispatches it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the tool implementations are in-process stubs and the agent's tool calls never leave the JVM.

## Generate the system

```sh
cp -r ./single-agent.general.agentcore-managed-harness  ~/my-projects/managed-harness-agent
cd ~/my-projects/managed-harness-agent
```

(Optional) Edit `SPEC.md` to swap in a different tool set or tighten the allowlist (e.g., restrict `query_metrics` to read-only namespaces only, or add a `run_query` tool for a database-connected deployment).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OperationsAgent** — an AutonomousAgent that accepts a natural-language question and a declared tool list; the managed runtime executes the tool-calling loop until the agent emits a final answer.
- **QueryWorkflow** — a two-step Workflow: `runStep` calls the agent and collects the answer; `recordStep` writes it to the entity.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle (SUBMITTED → RUNNING → ANSWERED / FAILED).
- **ToolAllowlistGuardrail** — the `before-tool-invocation` hook that checks every tool call against the configured allowlist before dispatch.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded question set in `src/main/resources/sample-events/seed-questions.jsonl`.
- `SPEC.md §5` — extend `QueryAnswer` with domain-specific fields (e.g., `confidenceLevel`, `sourceNamespace`, `citedTools`).
- `prompts/operations-agent.md` — narrow the agent's persona (an SRE deployer would scope it to production dashboards only; a data platform deployer would scope it to query planning).
- `eval-matrix.yaml` — wire a real tool registry by replacing the in-process stub allowlist with a lookup against a central policy store.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question → the agent calls tools → the answer appears in the UI with a tool-call trace.
2. The agent attempts to call a tool not in the allowlist → the `before-tool-invocation` guardrail blocks it → the agent is notified → the agent selects an allowed tool and continues.
3. The answer's tool-call trace is visible in the UI; every cited tool name matches the declared allowlist.
4. A question whose answer requires more than one tool call produces a trace with multiple entries, each stamped with the tool name and the output summary.

## License

Apache 2.0.
