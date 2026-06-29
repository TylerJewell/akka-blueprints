# Akka Sample: Inspect Multi-Agent Run (Phoenix)

A manager CodeAgent delegates search tasks to a search agent (ToolCallingAgent equipped with WebSearch and VisitWebpage tools), while OpenInference/Phoenix instrumentation captures token usage and per-step timing across the run. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance and full observability.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: **none**. This blueprint runs out of the box — web-search behaviour and Phoenix trace ingestion are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.traced-manager-search  ~/my-projects/traced-manager-search
cd ~/my-projects/traced-manager-search
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or URL allow-list.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ManagerAgent** — AutonomousAgent that decomposes a query into a search plan and synthesises results.
- **SearchAgent** — AutonomousAgent that executes web search and page visits, returning a `SearchResultBundle`.
- **TraceInspector** — AutonomousAgent that analyses the completed Phoenix trace and produces a `TraceReport`.
- **RunWorkflow** — Workflow that fans the search work out, collects the trace, and calls ManagerAgent for synthesis.
- **RunEntity** — EventSourcedEntity holding the full run lifecycle.
- **RunView** — projection the UI streams via SSE.
- **RunEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the query topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `AgentRun` record fields (e.g., add `totalCost`).
- `prompts/search-agent.md` — narrow the URL allow-list to a specific domain.
- `eval-matrix.yaml` — tighten the before-tool-call guardrail allow-list or add a cost cap control.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query → run enters `DISPATCHED`, then `SEARCHING`, then `TRACED`.
2. Search agent visits a blocked URL → the before-tool-call guardrail rejects the call; run enters `BLOCKED_TOOL` without calling the URL.
3. Eval-event sampling captures one manager synthesis decision and surfaces a quality score on the App UI.
4. Phoenix trace summary shows per-step token counts and wall-clock durations.

## License

Apache 2.0.
