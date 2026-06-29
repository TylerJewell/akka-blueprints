# Akka Sample: Sector Market Researcher

A continuous background worker ingests simulated news items, SEC-filing summaries, and broker research notes; classifies each for credit and risk relevance; synthesizes the relevant items into structured flags; and validates every flag output before it reaches a downstream desk. Demonstrates the **continuous-monitor** coordination pattern wired with a before-agent-response guardrail and a daily eval sampler.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.finance-analysis.market-researcher  ~/my-projects/market-researcher
cd ~/my-projects/market-researcher
```

(Optional) Edit `SPEC.md` to point the `ResearchPoller` at a real feed — a news API, an SEC EDGAR endpoint, or a broker note repository — or keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchPoller** — TimedAction firing every 20 s that drips simulated research items into `ResearchItemQueue`.
- **ResearchItemQueue** — EventSourcedEntity acting as an append-only audit log of raw inbound items.
- **RelevanceClassifierAgent** — typed Agent that classifies each item as `CREDIT_RELEVANT`, `RISK_RELEVANT`, `MARKET_COLOR`, or `NOISE`.
- **SynthesisAgent** — AutonomousAgent that synthesizes relevant items into structured `ResearchFlag` outputs.
- **ResearchWorkflow** — per-item Workflow orchestrating: classify → (if relevant) synthesize → validate output → deliver.
- **ResearchItemEntity** — EventSourcedEntity holding each item's full lifecycle.
- **ResearchView + ResearchEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **EvalRunner** — TimedAction firing once daily; samples delivered flags, scores flag precision/recall against seeded ground-truth labels.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated feed for a real news or filings source.
- `SPEC.md §5` — extend `ResearchItem` with desk-specific fields (`sector`, `issuerId`, `cusip`, etc.).
- `prompts/relevance-classifier.md` — narrow the classification to a specific sector or asset class.
- `eval-matrix.yaml` — replace the seeded ground-truth labels with analyst-labelled data under the eval-periodic mechanism.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated research item arrives → classified → synthesized → flag output guardrail passes → delivered to the desk queue.
2. An item classified as NOISE terminates without synthesis — no LLM synthesis call for noise items.
3. A flag that fails guardrail validation (missing source reference or confidence level) is rejected and the item transitions to REVIEW_REQUIRED.
4. EvalRunner scores at least one DELIVERED flag within the first evaluation cycle.

## License

Apache 2.0.
