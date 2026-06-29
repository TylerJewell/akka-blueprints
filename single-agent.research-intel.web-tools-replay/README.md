# Akka Sample: Server-Tool Web-Search Replay

A single research agent issues web-search tool calls against a provider API. Every tool call — query parameters, timestamps, and raw results — is recorded as a structured trace. A periodic replay harness re-issues the same queries and compares the current results against the recorded baseline, surfacing result-set drift across provider updates or ranking changes.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a periodic replay evaluator that detects behavioral drift in provider-side tool calls without relying on LLM judgment.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — web-search calls are simulated by a bundled stub that returns seeded result sets.

## Generate the system

```sh
cp -r ./single-agent.research-intel.web-tools-replay  ~/my-projects/web-tools-replay
cd ~/my-projects/web-tools-replay
```

(Optional) Edit `SPEC.md` to point at a different seed query library or adjust the drift-detection thresholds in the `ReplayDriftScorer`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SearchReplayAgent** — an AutonomousAgent that issues web-search tool calls for a given research query and returns a structured `SearchTrace` of every call made.
- **TraceRecordingConsumer** — a Consumer that subscribes to `SearchCompleted` events and writes the full trace to durable storage via `SearchRunEntity`.
- **ReplayWorkflow** — orchestrates one replay cycle: fetch the recorded trace → re-issue each search call → score drift.
- **SearchRunEntity** — an EventSourcedEntity holding the per-run lifecycle from submission through trace recording and replay verdict.
- **ReplayDriftScorer** — deterministic, rule-based scorer (no LLM call) that compares original and replayed result sets and emits a drift report.
- **SearchRunView + SearchRunEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded query library for your own topic area.
- `SPEC.md §5` — extend `WebSearchCall` with provider-specific metadata (e.g., `safeSearch`, `freshness`, `market` locale).
- `prompts/search-replay-agent.md` — narrow the agent's tool-calling strategy (a news-monitoring deployer might constrain it to time-bounded queries; a competitive-research deployer might add structured topic filtering).
- `eval-matrix.yaml` — wire a real web-search provider by replacing the stub with an actual tool-call binding.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a research query → the agent issues tool calls → a full trace is recorded and visible in the UI.
2. A replay run is triggered against a recorded trace → the drift scorer produces a `DriftReport` and the UI shows the drift score.
3. A trace replayed against a modified stub (simulating provider-side change) produces a non-zero drift score and flags the run.
4. Drift score and rationale appear on the same UI card as the original trace within 5 s of replay completion.

## License

Apache 2.0.
