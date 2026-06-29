# Akka Sample: Economic Research Agent

A market-economics coordinator delegates data-gathering to a DataCollector and interpretation to an Economist in parallel, then merges their outputs into one unified economic analysis report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound request queue and the economic data tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.economic-research-team  ~/my-projects/economic-research-agent
cd ~/my-projects/economic-research-agent
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **EconomicsCoordinator** — AutonomousAgent that frames a market question into parallel work items and synthesises results into a final report.
- **DataCollector** — AutonomousAgent that gathers quantitative indicators (modelled with seeded tools).
- **Economist** — AutonomousAgent that interprets the indicators and proposes implications.
- **AnalysisWorkflow** — Workflow that fans the work out to DataCollector and Economist in parallel, then calls EconomicsCoordinator for synthesis.
- **AnalysisReportEntity** — EventSourcedEntity holding the full report lifecycle.
- **AnalysisView** — projection the UI streams via SSE.
- **AnalysisEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the market questions the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `AnalysisReport` record fields (e.g., add `confidenceScore`).
- `prompts/data-collector.md` — narrow the agent to a single asset class or data source.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real market-data API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a market question → report enters `FRAMING`, then `IN_PROGRESS`, then `PUBLISHED`.
2. Workers fail-fast → if either DataCollector or Economist times out, the report enters `DEGRADED` with whichever partial output exists.
3. The financial-content guardrail fires → report with a speculative investment claim enters `BLOCKED`.
4. Wait after a successful publication; the report row shows an eval score.

## License

Apache 2.0.
