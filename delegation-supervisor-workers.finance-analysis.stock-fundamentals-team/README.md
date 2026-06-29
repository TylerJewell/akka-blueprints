# Akka Sample: AI Team for Fundamental Stock Analysis

A stock analysis coordinator fans fundamental analysis work out to three specialist agents running **in parallel** — financials, news, and ratio analysis — then synthesises a research recommendation. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance for investment-adjacent output.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound ticker stream and analysis tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.stock-fundamentals-team  ~/my-projects/stock-analysis
cd ~/my-projects/stock-analysis
```

(Optional) Edit `SPEC.md` to change the model provider, ticker list, or recommendation schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AnalysisCoordinator** — AutonomousAgent that decomposes a ticker request into subtasks and synthesises results into a `StockRecommendation`.
- **FinancialsAgent** — AutonomousAgent that extracts key financial metrics from income statement and balance sheet data.
- **NewsAgent** — AutonomousAgent that summarises recent news sentiment relevant to the ticker.
- **RatioAgent** — AutonomousAgent that computes and interprets standard valuation and profitability ratios.
- **AnalysisWorkflow** — Workflow that fans the three subtasks out in parallel, joins results, and calls the Coordinator for synthesis.
- **StockReportEntity** — EventSourcedEntity holding the full report lifecycle.
- **StockReportView** — projection the UI streams via SSE.
- **AnalysisEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the ticker list the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust `StockReport` record fields (e.g., add `targetPrice`).
- `prompts/financials-agent.md` — narrow the agent to a specific accounting standard.
- `eval-matrix.yaml` — extend the eval-event control with a sector-specific rubric.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a ticker → report enters `QUEUED`, then `ANALYSING`, then `COMPLETE`.
2. Workers fail-fast → if any specialist agent times out, the report enters `DEGRADED` with whichever partial results returned.
3. The output guardrail blocks a recommendation that contains direct investment advice.
4. Sector sanitizer strips any non-public price-sensitive phrases before the recommendation is surfaced.
5. Eval-event sampling captures one synthesis decision and surfaces the factuality score on the App UI.

## License

Apache 2.0.
