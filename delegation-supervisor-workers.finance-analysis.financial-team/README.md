# Akka Sample: Financial Assistant (Multi-Agent)

A financial supervisor delegates market research to a `MarketResearcher`, portfolio planning to a `PortfolioPlanner`, and report drafting to a `ReportDrafter` — all running under a `FinancialCoordinator`. The coordinator synthesises their outputs into one consolidated financial report. Demonstrates the **delegation-supervisor-workers** coordination pattern with finance-sector governance controls.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound request stream and the financial-data tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.financial-team  ~/my-projects/financial-team
cd ~/my-projects/financial-team
```

(Optional) Edit `SPEC.md` to change the ticker symbols the simulator uses, the model provider, or the report output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FinancialCoordinator** — AutonomousAgent that decomposes a financial query into work items and synthesises the team's outputs into a consolidated report.
- **MarketResearcher** — AutonomousAgent that gathers market data and price context for a security or sector.
- **PortfolioPlanner** — AutonomousAgent that evaluates risk/return characteristics and proposes allocation guidance.
- **ReportDrafter** — AutonomousAgent that writes the investor-ready narrative from the coordinator's synthesis.
- **FinancialWorkflow** — Workflow that fans work out to the three specialists in parallel, then asks the Coordinator to synthesise, and runs a finance-sector sanitizer before the report is committed.
- **FinancialReportEntity** — EventSourcedEntity holding the full report lifecycle.
- **FinancialView** — projection the UI streams via SSE.
- **FinancialEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the ticker symbols or query types the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `FinancialReport` record fields (e.g., add `confidenceInterval`).
- `prompts/market-researcher.md` — narrow the agent to a specific asset class.
- `eval-matrix.yaml` — extend the sanitizer with additional sector-specific word lists.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a financial query → report enters `DRAFTING`, then `IN_REVIEW`, then `PUBLISHED`.
2. Workers fail-fast → if any specialist times out, the report enters `DEGRADED` with whichever partial outputs exist.
3. Finance-sector sanitizer strips or flags investment-advice language before the report is committed.
4. Eval-event sampling captures one synthesis decision and surfaces the score on the App UI.

## License

Apache 2.0.
