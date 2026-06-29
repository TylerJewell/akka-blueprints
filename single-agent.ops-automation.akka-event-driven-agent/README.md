# Akka Sample: Ambient Durable Agent on Pub/Sub

A `WeatherAgent` listens on a pub/sub topic (`weather.requests`) and handles each inbound message as an independent durable workflow execution. Every message spawns its own `WeatherWorkflow` instance, which validates the payload, scrubs any PII, calls the agent, and records the result — all without a user sitting at a browser waiting for a reply.

Demonstrates the **single-agent** coordination pattern in the ops-automation domain. Two governance mechanisms are wired around the agent: a `before-agent-invocation` guardrail that validates the incoming event payload before the agent ever runs, and a PII sanitizer that scrubs inbound topic messages of identifiers before the model sees them. Because the trigger is a pub/sub message rather than a direct HTTP call, both controls must run at the message boundary rather than at an API gateway.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint ships with an in-process pub/sub broker stub; no external message broker is required to run out of the box.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.akka-event-driven-agent  ~/my-projects/ambient-weather-agent
cd ~/my-projects/ambient-weather-agent
```

(Optional) Edit `SPEC.md` to change the topic name, swap out the seeded weather request payloads, or replace the `WeatherAgent` with an agent that handles a different ops domain (infrastructure alerts, deployment events, cost anomalies).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WeatherAgent** — an AutonomousAgent that receives a validated, sanitized weather request and returns a structured `WeatherReport`.
- **WeatherWorkflow** — one durable workflow per inbound topic message. Steps: `validateStep` → `sanitizeStep` → `forecastStep` → `recordStep`.
- **RequestEntity** — an EventSourcedEntity holding the per-request lifecycle (RECEIVED → VALIDATED → SANITIZED → FORECASTING → COMPLETED / FAILED).
- **TopicConsumer** — a Consumer subscribed to `weather.requests`; each message starts a new `WeatherWorkflow`.
- **PayloadGuardrail** — validates the topic message structure before the agent is invoked (the `before-agent-invocation` hook).
- **RequestSanitizer** — scrubs PII from inbound payloads before they reach the agent.
- **RequestView + RequestEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded topic payloads for your own ops domain (change `weather.requests` to `infra.alerts` or `deploy.events`).
- `SPEC.md §5` — extend `WeatherReport` with domain-specific fields (e.g., `alertSeverity`, `affectedRegion`, `remediationAction`).
- `prompts/weather-agent.md` — narrow the agent's role (a network-ops deployer would constrain it to anomaly triage; an SRE deployer to incident pre-classification).
- `eval-matrix.yaml` — wire a real PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A message published on `weather.requests` spawns a workflow, passes validation, is sanitized, the agent produces a `WeatherReport`, and the report is visible in the UI.
2. A malformed topic message (missing required fields) is rejected by the `before-agent-invocation` guardrail before the agent runs; no workflow transitions to FORECASTING for that message.
3. A payload containing PII strings reaches the sanitizer; the agent's task attachment contains only the redacted form; the entity audit log retains the raw.
4. Two messages published in rapid succession produce two independent workflow executions with no state cross-contamination.

## License

Apache 2.0.
