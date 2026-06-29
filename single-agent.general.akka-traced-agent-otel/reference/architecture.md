# Architecture — traced-agent-otel

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call with end-to-end tracing wired around it. `ConversationEndpoint` accepts a submission, writes a `PromptSubmitted` event onto `ConversationEntity`, and starts `TraceExportWorkflow`. The workflow's `agentStep` calls `TracedConversationAgent` — the single AutonomousAgent — with the prompt text as `TaskDef.instructions(...)`. Inside the agent, `SpanInstrumentor` wraps every internal hook (before/after LLM call, before/after tool call, before/after memory access) and constructs `SpanRecord` POJOs. These are returned as part of `AgentResponse.spans`.

Once the agent returns, `SpanCollector` (a Consumer subscribed to `AgentRunCompleted` events) aggregates the raw span list into a `SpanSummary` and writes it back to the entity. Concurrently, the workflow's `spanFlushStep` calls `OtlpExporter.flush(spans)` and records the result — including whether the exporter was reachable and how many spans were dropped. The `monitorStep` then calls `PerformanceMonitor.score(spanSummary)` to produce a 1–5 quality score. `ConversationView` projects every entity event into a read-model row; `ConversationEndpoint` serves the read model over REST and SSE.

The graph has no second agent. `SpanCollector`, `PerformanceMonitor`, and `OtlpExporter` are non-LLM components — they process data, not generate it. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three notable moments:

1. `AgentRunCompleted` triggers `SpanCollector` asynchronously. The consumer runs independently of the workflow steps; `attachSpanSummary` may land before or after `spanFlushStep` finishes depending on timing.
2. `spanFlushStep` is non-blocking on export failure. If `OtlpExporter` returns an error, the workflow advances to `monitorStep` regardless, writing a `FlushResult` with `exporterHealthy = false`.
3. `monitorStep` reads `SpanSummary` synchronously. Because `SpanCollector` may not have run yet at this point, the monitor reads the raw `AgentResponse.spans` directly from the entity as a fallback. The `SpanSummary` from the consumer provides richer aggregates if available.

## State machine

Six states. The meaningful paths:

- The happy path is `SUBMITTED → RUNNING → COMPLETED → FLUSHED → MONITORED`.
- Two failure transitions land in `FAILED`: an agent error (or max-iterations exhausted) during `RUNNING`, and a `spanFlushStep` failure after two retries during `COMPLETED`. A `FAILED` conversation's prior span data is preserved on the entity.
- There is no `APPROVED` or `REJECTED` state. The agent produces answers and traces; it does not gate any downstream action. The blueprint deliberately stops at `MONITORED`.

## Entity model

`ConversationEntity` is the source of truth. It emits seven event types. `ConversationView` projects every event into a row. `SpanCollector` subscribes to entity events to compute the span summary. `TraceExportWorkflow` both reads (`getConversation`) and writes (`markRunning`, `recordAgentResponse`, `recordFlush`, `recordMonitorResult`, `fail`) on the entity. The relationship between `TracedConversationAgent` and `AgentResponse` is "returns" — the agent's task result is the response record including the span list.

## Observability governance flow

For any conversation that reaches `MONITORED`, the span data passed through:

1. **SpanInstrumentor** — in-agent wrapping of every LLM and tool hook; produces timestamped `SpanRecord` POJOs.
2. **OtlpExporter** — exports the span list over OTLP; records export health as `FlushResult`.
3. **SpanCollector** — aggregates raw spans into a `SpanSummary` with computed p-value inputs.
4. **PerformanceMonitor** — deterministic quality score surfacing instrumentation gaps, latency anomalies, and missing spans before a human has to read the raw trace.

Each step is independent. A broken exporter does not prevent monitoring; a missing span summary does not prevent flushing. The deployer sees the health of each step separately in the event log.
