# Akka Sample: Weather Agent

A single weather-query agent that answers natural-language weather questions by calling two external tools: a geocoding API that resolves a place name to coordinates, and a weather API that fetches current conditions or forecasts for those coordinates. The agent decides which tools to call, in what order, and how many times — the canonical multi-tool ReAct loop.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that validates API parameters (location string, date range, unit system) before the agent issues any external call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the geocoding and weather API calls are simulated by a bundled stub server; a deployer can swap in real endpoints by changing two config properties.

## Generate the system

```sh
cp -r ./single-agent.general.weather-tool-agent  ~/my-projects/weather-agent
cd ~/my-projects/weather-agent
```

(Optional) Edit `SPEC.md` to point at a real geocoding or weather API endpoint (e.g., OpenStreetMap Nominatim for geocoding, Open-Meteo for weather) by updating the `tool-endpoints` block in `application.conf`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WeatherAgent** — an AutonomousAgent that accepts a weather question as a task and calls the geocoding and weather tools as needed to produce a typed `WeatherAnswer`.
- **QueryWorkflow** — orchestrates validate → query → format per submitted question.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **ToolCallGuardrail** — validates every tool-call parameter set before the agent issues the call.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded example questions for your own deployment context (e.g., questions phrased in a specific locale or using internal city-name aliases).
- `SPEC.md §5` — extend `WeatherAnswer` with additional fields (e.g., `airQualityIndex`, `uvIndex`, `alertMessages`).
- `prompts/weather-agent.md` — restrict the agent's tool usage (e.g., disallow forecast calls beyond 7 days, require metric units by default).
- `eval-matrix.yaml` — replace the stub tool endpoints with real ones by referencing them in the guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user asks "What is the weather in Tokyo?" → the agent geocodes Tokyo, fetches current conditions, and returns a well-formed `WeatherAnswer`.
2. The agent attempts a tool call with a missing location parameter → the `before-tool-call` guardrail rejects it → the agent retries with a valid parameter set → a result lands.
3. A query for a location that fails geocoding transitions the entity to `FAILED` with a clear reason.
4. Every answered query is visible in the live list with the tool-call sequence recorded.

## License

Apache 2.0.
