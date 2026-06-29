# Akka Sample: Earth Engine Geospatial Analyst

A single geospatial-analyst agent receives a natural-language research query and a bounding region, queries Earth Engine raster and vector datasets (passed as a task attachment, never as inline prompt text), and returns a structured `AnalysisReport` — findings, anomalies, and recommended follow-up observations per requested indicator.

Demonstrates the **single-agent** coordination pattern wired in the research-intelligence domain. One `GeospatialAnalystAgent` carries the interpretation decision; surrounding components handle query ingestion, dataset retrieval, coordinate-safety validation, and on-decision scoring.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — satellite datasets are bundled as seeded JSONL fixtures and the agent's tool calls are simulated in-process.

## Generate the system

```sh
cp -r ./single-agent.research-intel.geospatial-analyst  ~/my-projects/geospatial-analyst
cd ~/my-projects/geospatial-analyst
```

(Optional) Edit `SPEC.md` to point at a different indicator catalogue (e.g., switch from the seeded vegetation / surface-temperature / flood-extent set to a custom land-cover or emissions set).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeospatialAnalystAgent** — an AutonomousAgent that accepts a spatial query and a dataset attachment, then returns a typed `AnalysisReport`.
- **AnalysisWorkflow** — orchestrates dataset-fetch → coordinate-check → analysis → eval per submitted query.
- **AnalysisEntity** — an EventSourcedEntity holding the per-analysis lifecycle.
- **DatasetFetcher** — a Consumer that subscribes to `QuerySubmitted` events, retrieves indicator snapshots for the requested bounding box, and emits `DatasetReady` back to the entity.
- **AnalysisView + AnalysisEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded indicator catalogue for your own (the JSONL file under `src/main/resources/sample-events/indicators.jsonl` after generation).
- `SPEC.md §5` — extend `AnalysisReport` with domain-specific fields (e.g., `confidenceInterval`, `temporalRange`, `spatialResolutionMeters`).
- `prompts/geospatial-analyst.md` — narrow the agent's role (a climate deployer would constrain it to IPCC AR6 indicator definitions; a disaster-response deployer to flood-extent and damage proxies).
- `eval-matrix.yaml` — wire a real coordinate-bounds validator by naming it under the bounds-check mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a spatial query → dataset is fetched → the agent analyses it → the report appears in the UI within 30 s.
2. The agent returns a malformed report on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed report lands.
3. Every recorded report has an on-decision eval score visible on the same UI card.
4. A query whose bounding box exceeds the configured maximum area is blocked before the agent call — the entity transitions to `FAILED` with a clear reason.

## License

Apache 2.0.
