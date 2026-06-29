# Akka Sample: Agent From Any LLM

A single tool-calling agent accepts a natural-language weather query, selects and calls the `get_weather` tool, and returns a structured `WeatherReport`. The blueprint demonstrates swapping the model backend — InferenceClient, Transformers (local), Ollama, LiteLLM, or OpenAI — without changing any application code, because the binding is expressed entirely in `application.conf`.

One governance control is wired: a `before-tool-call` guardrail validates every `get_weather` invocation for input sanity before the call leaves the agent loop.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The weather lookup is an in-process stub — no external API key or network connectivity required for the tool call itself.

## Generate the system

```sh
cp -r ./single-agent.general.any-llm-tool-agent  ~/my-projects/any-llm-tool-agent
cd ~/my-projects/any-llm-tool-agent
```

(Optional) Edit `SPEC.md` to replace the `get_weather` stub with a real weather-API call, add more tools, or constrain the agent to a different task domain.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WeatherAgent** — an AutonomousAgent wired with the `get_weather` tool and a `before-tool-call` guardrail that validates location strings before any tool invocation.
- **QueryWorkflow** — one workflow per submitted query: `invokeStep` calls the agent, `recordStep` writes the result.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle: submitted → invoking → result recorded → failed.
- **WeatherTools** — a plain Java class with the `get_weather` method (returns a stub `WeatherData` record); replaced by a live HTTP call when deployed against a real provider.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded example queries for your own (the JSONL file under `src/main/resources/sample-events/seed-queries.jsonl` after generation).
- `SPEC.md §5` — extend `WeatherData` with real provider fields (hourly forecast, UV index, wind speed).
- `prompts/weather-agent.md` — tighten the agent's scope (restrict to a set of known cities, or disallow queries that name private addresses).
- `eval-matrix.yaml` — replace the stub `WeatherTools.get_weather` with a live HTTP client by updating the `implementation` paragraph for G1.
- `application.conf` — switch `model-provider` to `ollama`, `litellm`, or `transformers` without touching any Java source.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a weather query → the agent invokes `get_weather` → a `WeatherReport` appears in the UI within 30 s.
2. A query with a blank or whitespace-only location string triggers the `before-tool-call` guardrail — the tool call is rejected, the agent produces a clarification request instead of a partial report.
3. The agent backend is toggled between two configured providers; both produce well-formed `WeatherReport` results for the same query.
4. The live SSE stream in the App UI reflects every status transition without a page refresh.

## License

Apache 2.0.
