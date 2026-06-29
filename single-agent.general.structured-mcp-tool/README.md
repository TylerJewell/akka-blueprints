# Akka Sample: Structured Output Tool (MCP)

A single CodeAgent calls an inline MCP server — built with FastMCP — that exposes a `get_weather_info` tool. The tool returns a Pydantic-validated, schema-typed weather payload. An `after-tool-call` guardrail validates that payload's structure before the agent ever consumes the data for its final response.

Demonstrates the **single-agent** coordination pattern in the general domain. One `WeatherQueryAgent` (AutonomousAgent) makes every decision; the surrounding components route requests, validate tool outputs, and expose read models for the UI.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The inline MCP server runs in-process; no external weather API is required.

## Generate the system

```sh
cp -r ./single-agent.general.structured-mcp-tool  ~/my-projects/structured-mcp-tool
cd ~/my-projects/structured-mcp-tool
```

(Optional) Edit `SPEC.md` to replace the `get_weather_info` tool with your own domain tool (e.g., a product-lookup tool or a database query tool) while preserving the `after-tool-call` guardrail pattern.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WeatherQueryAgent** — an AutonomousAgent that accepts a location query, calls `get_weather_info` on the inline MCP server, and returns a typed `WeatherSummary`.
- **WeatherQueryWorkflow** — orchestrates query submission → agent execution → result recording per request.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **McpToolResultGuardrail** — an `after-tool-call` guardrail that validates the tool response against the `WeatherPayload` schema before the agent processes it.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the seeded location examples with a different set of query examples.
- `SPEC.md §5` — extend `WeatherPayload` or `WeatherSummary` with additional fields (e.g., `uvIndex`, `windSpeed`, `airQualityIndex`).
- `prompts/weather-query-agent.md` — narrow the agent role to a specific downstream use (e.g., an agriculture deployer would instruct it to annotate weather data with crop-risk levels).
- `eval-matrix.yaml` — tighten the guardrail's schema check or add a second control.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a location query → the agent calls `get_weather_info` → the guardrail validates the payload → a `WeatherSummary` appears in the UI.
2. The MCP tool returns a payload missing a required field → the `after-tool-call` guardrail rejects it → the agent retries the tool call → a well-formed payload lands.
3. A query whose tool call succeeds but whose temperature field is outside the plausible range triggers a guardrail rejection with a clear rejection code.
4. The SSE stream delivers one event per query state transition; a late-joining client receives the full current row and needs no replay.

## License

Apache 2.0.
