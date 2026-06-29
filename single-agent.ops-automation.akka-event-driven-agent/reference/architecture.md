# Architecture — ambient-durable-agent-pubsub

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call, reached only after two successive gates. Messages arrive on the in-process `weather.requests` topic. `TopicConsumer` subscribes, records each arrival on `RequestEntity`, and starts a `WeatherWorkflow` instance per message. The workflow is the control plane: its `validateStep` calls `PayloadGuardrail` before the agent is ever mentioned; `sanitizeStep` calls `RequestSanitizer`; `forecastStep` — and only `forecastStep` — calls `WeatherAgent` with the sanitized payload as a task attachment. The result lands back on `RequestEntity` via `recordReport()`. `RequestView` projects every entity event into a UI-facing read model; `RequestEndpoint` serves the list and SSE stream to the browser.

There is no second agent. `PayloadGuardrail` and `RequestSanitizer` are plain Java classes that run synchronously inside workflow steps. That is what keeps this blueprint a faithful **single-agent** example — exactly one component talks to a model, and two deterministic gates stand in front of it.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct handoff points determine end-to-end latency:

1. The topic-delivery hop from `RequestEndpoint.publish()` to `TopicConsumer.onMessage()` — sub-millisecond for the in-process broker.
2. The workflow step progression — `validateStep` and `sanitizeStep` run synchronously in milliseconds each; the transition to `forecastStep` is the first point where latency accumulates.
3. The LLM round-trip inside `forecastStep` — bounded by the 60 s `stepTimeout`. For the mock provider this is microseconds; for a real provider typically 2–10 s.

The state updates visible in the UI arrive via SSE from `RequestView`'s stream-updates. A client that connects after the fact sees the current row state immediately on subscribe.

## State machine

Six states. The key invariant is that FORECASTING is unreachable if either gate rejected the message:

- `RECEIVED → VALIDATED`: `PayloadGuardrail` returned `valid = true`.
- `VALIDATED → SANITIZED`: `RequestSanitizer` produced a `SanitizedPayload`.
- `SANITIZED → FORECASTING`: `WeatherWorkflow.forecastStep` started the agent task.
- `FORECASTING → COMPLETED`: `WeatherAgent` returned a `WeatherReport`.
- `RECEIVED → FAILED`: `PayloadGuardrail` returned `valid = false` (invalid payload).
- `FORECASTING → FAILED`: agent task failed or timed out.

There is no `ACKNOWLEDGED` or `ACTED_ON` state. The report is observable; whether ops personnel act on it is outside the system boundary.

## Entity model

`RequestEntity` is the source of truth. It emits six event types covering the full lifecycle. `RequestView` projects every event into a read-model row. `TopicConsumer` only writes to the entity (via `receive()`) and starts the workflow — it never reads back. `WeatherWorkflow` both reads (`getRequest` inside `forecastStep` to retrieve the sanitized payload) and writes (`markValidated`, `attachSanitized`, `markForecasting`, `recordReport`, `complete`, `fail`). `WeatherAgent`'s relationship with `WeatherReport` is "returns" — the agent's task result becomes the record persisted by `recordReport`.

## Defence-in-depth governance flow

For any report that lands in the entity log, the message passed through:

1. **PayloadGuardrail** — structure and injection check; the agent is never invoked for invalid messages.
2. **RequestSanitizer** — PII scrub; the model never sees raw contact details or network identifiers.
3. **WeatherAgent** — one model call, one structured output.

Each gate is independent: removing the guardrail opens a prompt-injection path; removing the sanitizer opens a PII-leakage path. Neither absence is compensated by the other.
