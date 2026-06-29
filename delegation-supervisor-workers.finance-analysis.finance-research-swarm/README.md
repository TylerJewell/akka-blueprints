# Akka Sample: Finance Assistant Swarm Agent

A lead finance agent delegates data-gathering to four specialist workers — ticker resolver, price analyst, news analyst, and fundamentals analyst — running in parallel, then merges their outputs into one integrated stock report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance including an output guardrail, a sector sanitizer, and a decision-time eval event.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the market data tools and news feeds are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.finance-research-swarm  ~/my-projects/finance-research-swarm
cd ~/my-projects/finance-research-swarm
```

(Optional) Edit `SPEC.md` to change the ticker symbols in the simulator, the model provider, or the output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReportCoordinator** — AutonomousAgent that decomposes a stock query into parallel work items and merges all worker outputs into one integrated stock report.
- **TickerResolver** — AutonomousAgent that normalises a company name or partial ticker into a canonical ticker symbol and exchange.
- **PriceAnalyst** — AutonomousAgent that retrieves and interprets recent price data (modelled with seeded tools).
- **NewsAnalyst** — AutonomousAgent that retrieves and summarises recent news headlines (modelled with seeded tools).
- **FundamentalsAnalyst** — AutonomousAgent that retrieves and interprets key fundamental metrics (modelled with seeded tools).
- **ReportWorkflow** — Workflow that fans work out to the four specialists in parallel, then calls the Coordinator for synthesis with an output guardrail and sector sanitizer.
- **StockReportEntity** — EventSourcedEntity holding the full report lifecycle.
- **StockReportView** — projection the UI streams via SSE.
- **ReportEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the ticker symbols the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — add `peRatio` or `dividendYield` to the `FundamentalsSnapshot` record.
- `prompts/price-analyst.md` — narrow price interpretation to a specific time window.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real market-data API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a ticker query → report enters `RESOLVING`, then `IN_PROGRESS`, then `PUBLISHED`. UI reflects each transition via SSE.
2. A specialist worker times out → report enters `PARTIAL` with whichever partial outputs arrived; the summary notes the missing side.
3. The sector sanitizer strips a non-compliant recommendation phrase; the eval-event sampler scores the numeric accuracy of the merged report.

## License

Apache 2.0.
