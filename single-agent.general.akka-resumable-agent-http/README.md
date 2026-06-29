# Akka Sample: Durable Agent over HTTP with Crash-Resume

A single-agent system exposes two HTTP endpoints: `POST /agent/run` to start an agent run and `GET /agent/instances/{id}` to poll its status. The agent calls a `SlowWeatherTool` during execution, which takes long enough that a developer can kill the process mid-run, restart it, and observe the workflow resuming from the last recorded checkpoint — no work is lost.

Demonstrates the **single-agent** coordination pattern in the general domain. A `before-tool-call` guardrail gates every external tool invocation, and an on-incident evaluator records a structured incident event whenever the process crashes and resumes.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The `SlowWeatherTool` is simulated in-process; no external weather API is called.

## Generate the system

```sh
cp -r ./single-agent.general.akka-resumable-agent-http  ~/my-projects/akka-resumable-agent-http
cd ~/my-projects/akka-resumable-agent-http
```

(Optional) Edit `SPEC.md` to adjust the simulated tool latency (`SlowWeatherTool.DELAY_SECONDS`) or to swap the weather query for a different slow tool call that fits your domain.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WeatherQueryAgent** — an AutonomousAgent that accepts a location query, calls the `SlowWeatherTool` (configurable delay, default 8 s), and returns a `WeatherReport`.
- **AgentRunWorkflow** — orchestrates the per-run lifecycle: `initStep` → `toolCallStep` → `reportStep`. Checkpoints after each step; crash-and-restart resumes from the last completed step.
- **AgentRunEntity** — an EventSourcedEntity holding per-run state through its lifecycle.
- **ToolCallGuardrail** — a `before-tool-call` guardrail that validates the agent's tool invocation parameters before the `SlowWeatherTool` executes.
- **IncidentEvaluator** — an on-incident evaluator that fires after a crash/resume event is detected, records a structured `IncidentEvent`, and emits an `IncidentRecorded` audit log entry.
- **AgentRunView + AgentEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust how many example queries are seeded and what the `SlowWeatherTool` returns in mock mode.
- `SPEC.md §5` — extend `WeatherReport` with additional fields (temperature units, forecast horizon, air-quality index).
- `prompts/weather-query-agent.md` — narrow the agent's role (a deployer could restrict it to a specific region or constrain the output format).
- `eval-matrix.yaml` — wire the `before-tool-call` guardrail to a real allow-list (e.g., restrict the location argument to a known-safe set of city names).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user starts a run via `POST /agent/run`; the workflow completes and the report appears in the UI.
2. A user kills the process while `SlowWeatherTool` is executing; on restart the workflow resumes from `toolCallStep` and the report lands without re-running completed steps.
3. A malformed tool invocation (invalid location argument) is blocked by the `before-tool-call` guardrail; the agent retries with a corrected argument.
4. Every crash-resume event produces a visible `IncidentEvent` record in the UI's incident panel.

## License

Apache 2.0.
