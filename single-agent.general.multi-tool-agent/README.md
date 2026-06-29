# Akka Sample: Multiple Tools Agent

A single agent equipped with multiple external-API tools handles user requests that span weather look-up, currency conversion, and unit transformation in one conversational turn. Each tool call is gated by an input-validation guardrail that runs before the agent dispatches to any external endpoint.

Demonstrates the **single-agent** coordination pattern in the general domain. One `ToolCallingAgent` (AutonomousAgent) decides which tools to invoke and in what order; the surrounding components only prepare its context, enforce call safety, and track the execution history.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. All external API calls are simulated in-process; the blueprint runs out of the box.

## Generate the system

```sh
cp -r ./single-agent.general.multi-tool-agent  ~/my-projects/multi-tool-agent
cd ~/my-projects/multi-tool-agent
```

(Optional) Edit `SPEC.md` to add or replace tools — for example, swap the currency-conversion tool for a stock-quote tool, or add a timezone-lookup tool alongside the existing set.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ToolCallingAgent** — an AutonomousAgent that picks among registered tools, calls them in sequence, and returns a `ToolResponse` once the request is satisfied.
- **SessionWorkflow** — orchestrates validate-wait → dispatch → summarize per submitted request.
- **SessionEntity** — an EventSourcedEntity holding the per-session request and tool-call history.
- **ToolCallGuardrail** — a `before-tool-call` guardrail that validates tool inputs before any external API is reached.
- **WeatherTool**, **CurrencyTool**, **UnitTool** — three simulated external-API adapters registered as `@tool` capabilities on the agent.
- **SessionView + SessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the seeded request examples or change tool names to match your environment.
- `SPEC.md §5` — extend `ToolCallRecord` with additional metadata fields (e.g., latency, external request id, cache-hit flag).
- `prompts/tool-calling-agent.md` — narrow or broaden the agent's tool-selection strategy; add priority rules between tools.
- `eval-matrix.yaml` — replace the simulated tool adapters with real HTTP clients by naming them under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a multi-part request → the agent calls multiple tools in sequence → a combined answer appears in the UI with one `ToolCallRecord` per tool invoked.
2. A request containing an invalid currency code is submitted → the `before-tool-call` guardrail rejects the currency tool call → the agent reformulates and retries → a valid answer lands.
3. A request that exhausts the agent's iteration budget without resolving the query lands in `FAILED` state; the partial tool-call history is visible in the UI.
4. Every session shows the per-tool call log; the timestamp, input arguments, and result for each call are visible in the session detail pane.

## License

Apache 2.0.
