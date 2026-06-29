# Akka Sample: Multi-Agent Supply Chain Optimizer

A demand-signal coordinator delegates inventory analysis to a StockAnalyst and route optimization to a LogisticsPlanner running **in parallel**, then merges their outputs into a unified supply chain recommendation. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound demand signal stream and the optimization tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.ops-automation.supply-chain-team  ~/my-projects/supply-chain-team
cd ~/my-projects/supply-chain-team
```

(Optional) Edit `SPEC.md` to change the demand signal topics the simulator drips, the model provider, or the recommendation output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DemandCoordinator** — AutonomousAgent that decomposes a demand signal into parallel work items and synthesises the final supply chain recommendation.
- **StockAnalyst** — AutonomousAgent that evaluates inventory positions and stock-out risk.
- **LogisticsPlanner** — AutonomousAgent that proposes routing and replenishment schedules.
- **OptimizationWorkflow** — Workflow that fans work out to StockAnalyst and LogisticsPlanner in parallel, then calls DemandCoordinator for synthesis.
- **SupplyOrderEntity** — EventSourcedEntity holding the full recommendation lifecycle.
- **OrderQueueEntity** — EventSourcedEntity logging each submitted demand signal for replay.
- **SupplyOrderView** — projection the UI streams via SSE.
- **DemandSignalConsumer** — Consumer that starts one workflow per demand signal event.
- **DemandSimulator** — TimedAction that drips canned demand signals every 60 seconds.
- **FulfillmentEvalSampler** — TimedAction that scores completed recommendations every 5 minutes.
- **SupplyEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the demand signal topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `SupplyRecommendation` record fields (e.g., add `confidenceScore`).
- `prompts/stock-analyst.md` — narrow the analyst to a single product category.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real inventory API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a demand signal → order enters `PENDING`, then `ANALYZING`, then `RECOMMENDED`.
2. Workers fail-fast → if either StockAnalyst or LogisticsPlanner times out, the order enters `DEGRADED` with whichever partial output exists.
3. Eval-event sampling captures one recommendation decision and surfaces the score on the App UI.

## License

Apache 2.0.
