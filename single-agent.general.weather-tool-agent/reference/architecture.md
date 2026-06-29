# Architecture — weather-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question, writes a `QuestionSubmitted` event to `QueryEntity`, and immediately starts a `QueryWorkflow`. The workflow's `processStep` calls `WeatherAgent` — the single AutonomousAgent — with the question text as task instructions. The agent decides which tools to call: first `geocode(location)` to resolve coordinates, then `getWeather(lat, lon, units, forecastDays)` to fetch conditions or a forecast. Before each tool call exits the agent loop, `ToolCallGuardrail` runs parameter validation. Once all tool calls succeed, the agent assembles a `WeatherAnswer` and returns it. The workflow writes `AnswerRecorded` to `QueryEntity`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `WeatherToolProvider` is a synchronous stub with `@Tool`-annotated methods — it is a capability registered on the agent, not an independent decision-maker. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1): a current-conditions question for a known city. Two tool calls happen in sequence inside the agent loop:

1. `geocode("Paris")` — resolves to coordinates. The guardrail accepts the non-empty location string.
2. `getWeather(48.85, 2.35, "metric", 1)` — fetches today's conditions. The guardrail accepts the in-range parameters.

Both calls are recorded as `ToolCallRecorded` events on `QueryEntity`, giving the UI a tool-call timeline for the selected query. The agent then returns the `WeatherAnswer` in one shot; the workflow writes it and the entity transitions to `ANSWERED`.

## State machine

Four states. The paths:

- The happy path is `SUBMITTED → PROCESSING → ANSWERED`.
- One failure transition lands in `FAILED`: any unrecoverable error during `PROCESSING` — geocoding returning no results, the guardrail exhausting all 4 iterations, or an agent timeout. A `FAILED` query retains all `ToolCallRecord` entries emitted before the failure, so the UI can show how far the agent got.
- There is no `PENDING_REVIEW` state. Weather answers are informational; no human approval gate is required. The blueprint deliberately stops at `ANSWERED`.

## Entity model

`QueryEntity` is the source of truth. It emits five event types. `QueryView` projects every event into a row used by the UI. `QueryWorkflow` both reads (`getQuery`) and writes (`startProcessing`, `recordToolCall`, `recordAnswer`, `fail`) on the entity. The relationship between `WeatherAgent` and `WeatherAnswer` is "returns" — the agent's task result is the answer record. The relationship between `WeatherAgent` and `WeatherToolProvider` is "calls" — tool methods are invoked synchronously inside the agent loop.

## Defence in depth — parameter validation flow

For any answer that lands in the entity log, every tool call the agent made passed through:

1. **before-tool-call guardrail** — empty location strings, out-of-range forecast windows, invalid unit systems, and invalid coordinates are caught before any call exits the loop.
2. **WeatherAgent** — one model call, one structured output, up to 4 iterations.
3. **WeatherToolProvider** — stub or real API, called only with validated parameters.

The single check here is lighter than a multi-mechanism governance stack because weather queries carry low inherent risk — the main failure mode is a bad tool parameter producing a wrong answer, not a safety violation or data leak. The guardrail addresses that failure mode directly.
