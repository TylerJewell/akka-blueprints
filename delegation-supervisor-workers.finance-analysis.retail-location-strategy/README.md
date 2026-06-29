# Akka Sample: Retail AI Location Strategy

A location coordinator delegates site-scoring work to a Market Analyst and a Demographics Analyst running **in parallel**, then merges their outputs into one ranked location recommendation. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the candidate site stream and the scoring tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.retail-location-strategy  ~/my-projects/retail-location-strategy
cd ~/my-projects/retail-location-strategy
```

(Optional) Edit `SPEC.md` to change the candidate site simulator cadence, model provider, or scoring weights.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **LocationCoordinator** — AutonomousAgent that decomposes a candidate site into scoring tasks and synthesises parallel results into a ranked recommendation.
- **MarketAnalyst** — AutonomousAgent that evaluates trade-area market conditions for a candidate site.
- **DemographicsAnalyst** — AutonomousAgent that assesses population and consumer-profile fit for a candidate site.
- **LocationWorkflow** — Workflow that fans the work out to MarketAnalyst and DemographicsAnalyst in parallel, then calls LocationCoordinator for synthesis.
- **CandidateSiteEntity** — EventSourcedEntity holding the full site-evaluation lifecycle.
- **LocationView** — projection the UI streams via SSE.
- **LocationEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the candidate sites the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `SiteRecommendation` record fields (e.g., add `competitorProximityScore`).
- `prompts/market-analyst.md` — narrow the agent to a specific retail vertical.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real geodata API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a candidate site → evaluation enters `SCORING`, then `IN_PROGRESS`, then `RECOMMENDED` or `NOT_RECOMMENDED`.
2. Worker fail-fast — if either analyst times out, the evaluation enters `DEGRADED` with whichever partial score exists.
3. Eval-event sampling captures one recommendation decision and surfaces the score on the App UI.

## License

Apache 2.0.
