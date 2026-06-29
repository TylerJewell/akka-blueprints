# Akka Sample: Traced Agent

A durable agent that emits application-level OpenTelemetry spans for every LLM call, tool invocation, and memory operation, then exports them alongside Akka workflow spans to a Zipkin backend for end-to-end distributed tracing.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a periodic latency-and-quality monitor that reads the trace stream after each agent run, and a deployer-runtime monitor anchored to the live span export pipeline.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Zipkin is bundled as an optional in-process stub for local runs; the production path exports to any OTLP-compatible collector.

## Generate the system

```sh
cp -r ./single-agent.general.akka-traced-agent-otel  ~/my-projects/traced-agent
cd ~/my-projects/traced-agent
```

(Optional) Edit `SPEC.md` to point at a real Zipkin or Jaeger endpoint (update `akka.javasdk.tracing.endpoint` in `application.conf`), or swap the seeded prompts for your own use case.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TracedConversationAgent** — an AutonomousAgent that wraps every LLM turn, tool call, and memory read/write in an OpenTelemetry span before delegating to the model.
- **TraceExportWorkflow** — orchestrates prompt-wait → agent-call → span-flush steps with explicit per-step timeouts.
- **ConversationEntity** — an EventSourcedEntity holding per-conversation lifecycle and accumulated span metadata.
- **SpanCollector** — a Consumer that subscribes to `AgentRunCompleted` events, aggregates span records, and writes a `SpanSummary` back to the entity.
- **ConversationView + ConversationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded prompt set for your domain (the JSONL file under `src/main/resources/sample-events/seed-prompts.jsonl` after generation).
- `SPEC.md §5` — extend `SpanRecord` with domain-specific tags (e.g., `modelVersion`, `promptHash`, `costEstimate`).
- `prompts/traced-conversation-agent.md` — narrow the agent's role or add tool definitions relevant to your application.
- `eval-matrix.yaml` — point the periodic monitor at a real Prometheus or Grafana endpoint by updating the implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt → the agent runs → spans appear in the Zipkin UI within 5 s.
2. The periodic monitor fires after each run and emits a `PerformanceMonitorResult` with p50/p95 latency and a quality score.
3. The deployer-runtime monitor subscribes to span export events and records export health (success count, error count, last-export timestamp).
4. An agent run with a failing tool call still produces complete spans: the error span is exported alongside the successful ones, and the UI flags the card.

## License

Apache 2.0.
