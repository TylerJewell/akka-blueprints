# Akka Sample: Durable Weather Agent via Workflow API

A `WeatherAgent` runs as a Workflow-backed durable agent. Callers interact with it exclusively through the Workflow API — either `trigger_agent` (fire-and-wait: the caller blocks until the agent finishes) or `call_agent` (the agent runs as a child workflow spawned by the caller's own workflow). The agent fetches live weather data through typed tool calls; each tool call is an activity in the underlying workflow, so it survives crashes, restarts, and network blips.

Demonstrates the **single-agent** coordination pattern in a general-purpose domain, wired with one governance mechanism: a `before-tool-call` guardrail that gates every outbound HTTP call before the activity executes.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Weather API calls are simulated in-process during local development; no third-party account is required.

## Generate the system

```sh
cp -r ./single-agent.general.akka-durable-tool-agent  ~/my-projects/durable-weather-agent
cd ~/my-projects/durable-weather-agent
```

(Optional) Edit `SPEC.md` to point at a different set of weather tools (e.g., add a `get_air_quality` or `get_marine_forecast` tool) or swap the seeded city list.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WeatherAgent** — an AutonomousAgent that accepts a weather query and returns a typed `WeatherReport`. Tool calls are gated by a `before-tool-call` guardrail before any outbound HTTP activity fires.
- **AgentWorkflow** — a Workflow that wraps each agent invocation. Supports two entry points: `triggerAgent` (blocking, caller waits for the report) and `callAgent` (child-workflow mode, caller's workflow spawns this one).
- **WeatherJobEntity** — an EventSourcedEntity holding the per-job lifecycle: queued → running → completed / failed.
- **WeatherToolActivity** — a Workflow activity that executes each tool call; the guardrail runs here before the HTTP request fires.
- **JobView + JobEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded city queries for your own (the JSONL file under `src/main/resources/sample-events/seed-queries.jsonl` after generation).
- `SPEC.md §5` — extend `WeatherReport` with domain-specific fields (e.g., `alertLevel`, `forecastDays`, `precipitationMm`).
- `prompts/weather-agent.md` — narrow the agent's role (an aviation deployer would restrict it to METAR/TAF-format summaries; an agriculture deployer to soil-moisture and frost risk).
- `eval-matrix.yaml` — wire a real weather API (e.g., Open-Meteo or Tomorrow.io) by naming it in the `WeatherToolActivity` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a city query via `trigger_agent` → the job transitions through QUEUED → RUNNING → COMPLETED and the weather report appears in the UI within 30 s.
2. The `call_agent` path: a parent workflow spawns the WeatherAgent as a child, waits for the child to finish, and incorporates the child's report into its own result.
3. A tool call with a disallowed location pattern is blocked by the `before-tool-call` guardrail; the agent retries with a corrected argument or returns a partial report.
4. The service restarts mid-job; the workflow resumes from the last completed activity without re-running earlier tool calls.

## License

Apache 2.0.
