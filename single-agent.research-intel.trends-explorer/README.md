# Akka Sample: Google Trends Agent

A single agent retrieves trending search topics by region and time period, returning a structured `TrendReport` with ranked topics, breakout signals, and related query clusters. The region and time window travel into the agent as structured task parameters, not as inline prompt text.

Demonstrates the **single-agent** coordination pattern in the research-intelligence domain. One `TrendsExplorerAgent` (AutonomousAgent) owns the entire synthesis; surrounding components handle request lifecycle, result persistence, and a quality evaluator that scores every report for coverage and coherence.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — trend data is seeded in-process and the agent's synthesis is the only LLM call.

## Generate the system

```sh
cp -r ./single-agent.research-intel.trends-explorer  ~/my-projects/trends-explorer
cd ~/my-projects/trends-explorer
```

(Optional) Edit `SPEC.md` to point at different seeded trend datasets (e.g., switch from the default global/US/EU seed to region-specific seeds for APAC or LATAM).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TrendsExplorerAgent** — an AutonomousAgent that accepts a region + time-window task and synthesises a `TrendReport` from seeded trend data passed as a task attachment.
- **TrendRequestWorkflow** — orchestrates data-fetch → synthesize → eval per submitted request.
- **TrendRequestEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **TrendDataFetcher** — a Consumer that subscribes to `TrendQuerySubmitted` events, assembles the matching seed data into a `RawTrendPayload`, and emits `TrendDataReady` back to the entity.
- **TrendReportView + TrendEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded trend datasets for live data by wiring a real trends-data HTTP client in `TrendDataFetcher`.
- `SPEC.md §5` — extend `TrendReport` with domain-specific fields (e.g., `sentimentSignal`, `competitorMentions`, `mediaVolumeIndex`).
- `prompts/trends-explorer.md` — narrow the agent's role (a finance deployer would constrain it to ticker-adjacent trend detection; a media deployer to content virality signals).
- `eval-matrix.yaml` — add controls if you wire a real external data source that needs its own rate-limiting or attribution checks.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a region + time window → data is fetched → the agent synthesises a report → the report appears in the UI with ranked topics.
2. The agent returns a malformed report on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed report lands.
3. Every recorded report has an on-decision eval score visible on the same UI card.
4. A report whose topics carry empty `rationale` strings receives an eval score of 1 with a clear explanation; the UI flags the card.

## License

Apache 2.0.
