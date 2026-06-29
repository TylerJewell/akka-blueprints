# Akka Sample: Currency Agent

A single exchange-rate agent accepts a conversion request — source currency, target currency, and amount — and returns a typed `ConversionResult` with the converted amount, the exchange rate used, and a confidence note. Exchange rate data arrives as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern in the finance-analysis domain. One `ExchangeRateAgent` (AutonomousAgent) performs the conversion and narrative; the surrounding components fetch the rate data, validate the structured output, and score each conversion for rate-staleness risk.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — exchange rate data is bundled as a seeded JSONL file; the agent's tool calls are simulated in-process.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.currency-converter  ~/my-projects/currency-agent
cd ~/my-projects/currency-agent
```

(Optional) Edit `SPEC.md` to point at a live exchange-rate provider by replacing the seeded rate file reference in Section 3 with an HTTP fetch configuration.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ExchangeRateAgent** — an AutonomousAgent that accepts a conversion request (currencies + amount) and rate data as a task attachment, and returns a typed `ConversionResult`.
- **ConversionWorkflow** — orchestrates rate-fetch → convert → eval per submitted request.
- **ConversionEntity** — an EventSourcedEntity holding the per-conversion lifecycle.
- **RateFetcher** — a Consumer that subscribes to `ConversionRequested` events, loads the current rate snapshot, and emits `RateAttached` back to the entity.
- **ConversionView + ConversionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded rate file for a live rate provider (e.g., an ECB XML feed or an Open Exchange Rates endpoint) by naming the fetch URL in the RateFetcher block.
- `SPEC.md §5` — extend `ConversionResult` with deployment-specific fields (e.g., `regulatoryRegion`, `reportingCurrency`, `hedgingRecommendation`).
- `prompts/exchange-rate-agent.md` — narrow the agent's role (a treasury deployer might constrain it to G10 currencies only; a remittance deployer might require a corridor-specific fee schedule).
- `eval-matrix.yaml` — the baseline ships with no controls. Add controls here as your governance requirements become clear; the eval-matrix schema documents the available mechanism kinds.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a conversion request → the rate is fetched → the agent returns a well-formed `ConversionResult` visible in the UI.
2. The agent returns a malformed result on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed result lands.
3. The conversion timeline from request to result is visible per-card in the live list.
4. The seeded rate file covers the major currency pairs so the agent always has rate context for the bundled examples.

## License

Apache 2.0.
