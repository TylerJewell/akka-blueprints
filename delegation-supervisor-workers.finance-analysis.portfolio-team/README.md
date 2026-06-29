# Akka Sample: Portfolio Assistant (Multi-Agent)

A portfolio coordinator delegates holdings analysis and market-context gathering to two specialist agents running **in parallel**, then merges their outputs into one unified portfolio report. Demonstrates the **delegation-supervisor-workers** coordination pattern with sector-aware input sanitization and an investment-output guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the holdings stream and the market-data tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.portfolio-team  ~/my-projects/portfolio-team
cd ~/my-projects/portfolio-team
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PortfolioCoordinator** — AutonomousAgent that decomposes a holdings set into an analysis plan and consolidates worker reports.
- **HoldingsAnalyst** — AutonomousAgent that evaluates individual position metrics and sector exposure.
- **MarketContextAgent** — AutonomousAgent that gathers macro-economic and sector background relevant to the portfolio.
- **PortfolioWorkflow** — Workflow that fans work out to HoldingsAnalyst and MarketContextAgent in parallel, then calls PortfolioCoordinator for consolidation.
- **PortfolioReportEntity** — EventSourcedEntity holding the full report lifecycle.
- **PortfolioView** — projection the UI streams via SSE.
- **PortfolioEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the holdings the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `PortfolioReport` record fields (e.g., add `riskScore`).
- `prompts/coordinator.md` — narrow the coordinator to a specific asset class.
- `eval-matrix.yaml` — add a `before-tool-call` sanitizer if you wire a real market-data API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a holdings set → report enters `PLANNING`, then `IN_PROGRESS`, then `CONSOLIDATED`.
2. Workers fail-fast → if either HoldingsAnalyst or MarketContextAgent times out, the report enters `DEGRADED` with whichever partial output exists.
3. The sector sanitizer blocks a submission that contains a prohibited sector tag before analysis begins.
4. The output guardrail blocks a consolidated report that contains a direct buy/sell recommendation.

## License

Apache 2.0.
