# Akka Sample: Investment Research Multi-Agent

A report coordinator delegates fundamental data gathering to an Equity Analyst and market sentiment gathering to a Market Scout, both running **in parallel**, then merges their outputs into one investment research report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance, including a sector-scoped content sanitizer and a periodic eval sampler for numeric claim quality.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound request queue and the financial data tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.investment-research-team  ~/my-projects/investment-research
cd ~/my-projects/investment-research
```

(Optional) Edit `SPEC.md` to change the ticker symbols the simulator drips, or adjust the report schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReportCoordinator** — AutonomousAgent that decomposes a research request into sub-tasks and synthesises the final investment report.
- **EquityAnalyst** — AutonomousAgent that gathers fundamental financial data (earnings, ratios, guidance).
- **MarketScout** — AutonomousAgent that gathers market sentiment and recent price-action context.
- **ReportWorkflow** — Workflow that fans the work out to EquityAnalyst and MarketScout in parallel, then calls ReportCoordinator for synthesis.
- **ResearchReportEntity** — EventSourcedEntity holding the full report lifecycle.
- **ReportView** — projection the UI streams via SSE.
- **ReportEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the ticker symbols the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `InvestmentReport` record fields (e.g., add `priceTarget`).
- `prompts/equity-analyst.md` — narrow the analyst to a specific sector.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real market data API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a ticker and research request → report enters `PLANNING`, then `IN_PROGRESS`, then `PUBLISHED`.
2. Workers fail-fast → if either EquityAnalyst or MarketScout times out, the report enters `DEGRADED` with whichever partial output exists.
3. Sector sanitizer blocks regulated investment content that violates the sector guardrail.
4. Eval sampler captures one synthesis decision and surfaces a numeric claim quality score on the App UI.

## License

Apache 2.0.
