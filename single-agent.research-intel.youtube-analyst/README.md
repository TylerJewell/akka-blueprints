# Akka Sample: YouTube Analyst

A single channel-analyst agent fetches YouTube channel metrics and audience data, then returns a structured performance report: top-performing videos, engagement trends, audience demographics summary, and growth signals. Channel identifiers ride into the agent as task inputs, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern in the research-intel domain. One `ChannelAnalystAgent` (AutonomousAgent) carries the entire analysis decision; the surrounding components prepare its tool outputs and audit its conclusions. Two governance mechanisms are wired around the agent: a `before-agent-response` guardrail that validates the report's structure and data references, and an on-decision evaluator that scores every report for analytical completeness.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the YouTube data corpus lives in-process as seeded JSON fixtures and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.research-intel.youtube-analyst  ~/my-projects/youtube-analyst
cd ~/my-projects/youtube-analyst
```

(Optional) Edit `SPEC.md` to point at a different channel seed set (e.g., switch from the default consumer-brand fixtures to media-publisher or gaming-creator fixtures).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ChannelAnalystAgent** — an AutonomousAgent that accepts a channel analysis task and returns a typed `ChannelReport`.
- **AnalysisWorkflow** — orchestrates fetch-wait → analyse → eval per requested channel.
- **ChannelEntity** — an EventSourcedEntity holding the per-analysis lifecycle.
- **MetricsFetcher** — a Consumer that subscribes to `AnalysisRequested` events, retrieves seeded channel metrics and video data, and emits `MetricsFetched` back to the entity.
- **AnalysisView + AnalysisEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded channel fixtures for your own (the JSONL file under `src/main/resources/sample-events/channel-fixtures.jsonl` after generation).
- `SPEC.md §5` — extend `ChannelReport` with industry-specific fields (e.g., `competitorBenchmarks`, `monetizationTier`, `contentPillar`).
- `prompts/channel-analyst.md` — narrow the agent's role (a media deployer would focus it on engagement-per-impression; a brand deployer on share-of-voice and sentiment).
- `eval-matrix.yaml` — wire a real YouTube Data API v3 client by naming it under the metrics-fetcher implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user requests a channel analysis → metrics are fetched → the agent analyses them → the report appears in the UI.
2. The agent returns a malformed report on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed report lands.
3. Every recorded report has an on-decision eval score visible on the same UI card.
4. Reports whose trend claims are not backed by the fetched metrics receive an eval score of 1 with a clear rationale.

## License

Apache 2.0.
