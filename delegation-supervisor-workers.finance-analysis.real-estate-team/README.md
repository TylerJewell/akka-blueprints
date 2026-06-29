# Akka Sample: Real Estate Investment Multi-Agent

A deal coordinator delegates market research to a Market Specialist and financial modeling to a Finance Specialist running **in parallel**, then merges their outputs into one investment recommendation report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance controls for regulated investment analysis.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound deal queue and property data tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.real-estate-team  ~/my-projects/real-estate-team
cd ~/my-projects/real-estate-team
```

(Optional) Edit `SPEC.md` to change the deal queue topics the simulator drips, or to adjust the investment thresholds in the risk model.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DealCoordinator** — AutonomousAgent that decomposes a property deal into a market research query and a financial analysis question, then synthesises both outputs into a final investment recommendation.
- **MarketSpecialist** — AutonomousAgent that gathers comparable sales, neighborhood trends, and pricing data (modelled with seeded tools).
- **FinanceSpecialist** — AutonomousAgent that models cap rate, NOI, cash-on-cash return, and debt service coverage.
- **EvaluationWorkflow** — Workflow that fans work out to MarketSpecialist and FinanceSpecialist in parallel, then calls DealCoordinator for synthesis.
- **DealEntity** — EventSourcedEntity holding the deal's full lifecycle.
- **DealView** — projection the UI streams via SSE.
- **DealEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the property scenarios the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — add `debtServiceCoverageRatio` or `loanToValue` to the `FinancialModel` record.
- `prompts/market-specialist.md` — narrow the agent to a specific metropolitan market.
- `eval-matrix.yaml` — add a `before-tool-call` sanitizer if you wire a real MLS or pricing API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a property deal → recommendation enters `SCOPING`, then `EVALUATING`, then `RECOMMENDED`.
2. Workers fail-fast → if either MarketSpecialist or FinanceSpecialist times out, the recommendation enters `DEGRADED` with whichever partial output came back.
3. Sector sanitizer blocks a disallowed property type (e.g., cannabis dispensary); deal enters `REJECTED` before agents run.
4. Output guardrail blocks an unsafe recommendation (e.g., fabricated cap rate citations); deal enters `BLOCKED`.
5. Wait after a successful recommendation; the deal row shows an eval score from the periodic quality sampler.

## License

Apache 2.0.
