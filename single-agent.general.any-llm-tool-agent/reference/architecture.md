# Architecture — any-llm-tool-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one tool-calling LLM. `QueryEndpoint` accepts a query, writes a `QuerySubmitted` event onto `QueryEntity`, and immediately starts a `QueryWorkflow` instance. The workflow's `invokeStep` calls `WeatherAgent` — the single AutonomousAgent — with the query text as `TaskDef.instructions(...)`. The agent resolves a location string and issues a `get_weather` tool call; before the tool body executes, the `WeatherToolGuardrail` `before-tool-call` hook validates the location argument. If the argument passes, `WeatherTools.get_weather` returns a `WeatherData` stub; the agent wraps it in a `WeatherReport` and returns to the workflow. The workflow then calls `QueryEntity.recordReport`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `WeatherTools` is a plain tool class — it holds tool methods; it does not call a model. `WeatherToolGuardrail` is a hook class — it intercepts a call; it does not make a decision. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two moments of internal routing:

1. The workflow starts immediately after `QueryEndpoint` receives the POST; it does not wait for an event from a Consumer. This means the `invokeStep` begins in the same request cycle.
2. The agent's tool call passes through the `WeatherToolGuardrail` hook synchronously before `WeatherTools.get_weather` runs. The hook adds no I/O; its cost is a regex check.

The agent call is bounded by `invokeStep`'s 60 s timeout, which accommodates backend variation from a fast mock through a remote LLM. The `recordStep` is effectively instant — it writes one event.

## State machine

Four states. The interesting paths:

- The happy path is `SUBMITTED → INVOKING → RESULT_RECORDED`.
- Two failure transitions land in `FAILED`: a workflow start error during `SUBMITTED`, and an agent error (network, timeout, or guardrail-exhaustion) during `INVOKING`. A `FAILED` query retains its `WeatherQuery` on the entity — the UI shows the original query text with a red status pill.
- There is no `APPROVED` state. The weather report is informational; the user reads it and acts outside the system.

## Entity model

`QueryEntity` is the source of truth. It emits four event types. `QueryView` projects every event into a row used by the UI. `QueryWorkflow` both reads (`getQuery`) and writes (`markInvoking`, `recordReport`, `fail`) on the entity. The relationship between `WeatherAgent` and `WeatherReport` is "returns" — the agent's task result is the report record.

## Backend-agnostic wiring

The key educational property of this blueprint is that no Java source file branches on the model backend. The `requestedBackend` field stored on `WeatherQuery` is metadata for traceability — it appears in the UI and in the entity log — but the actual provider selection happens in `application.conf`. A deployer adds an `ollama` or `litellm` block to the config, sets `agent.default = ollama`, and the running system switches backends without touching `WeatherAgent.java`. The `WeatherToolGuardrail` and `WeatherTools` classes are equally backend-agnostic: they operate on the tool argument and return value, which are identical regardless of which model issued the call.
