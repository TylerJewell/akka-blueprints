# Architecture — workflow-observability

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`WorkloadPoller` is the heartbeat — a TimedAction that ticks every 20 s and writes simulated `WorkloadItemReceived` events into `WorkloadQueue` (event-sourced for audit). A `TraceCollector` Consumer subscribes to that queue, opens a root span, and emits `ItemRegistered` on the per-item `WorkloadItemEntity`. That registration starts an `ObservabilityWorkflow` instance, which orchestrates: route → (if SUMMARISE) process → export spans → done. The ESCALATE and SKIP branches terminate immediately without invoking `SummaryAgent`.

`TraceExporter` runs as a second Consumer, subscribing to `SpanEmitted` events on `WorkloadItemEntity`. It forwards each span to the configured observability backend (Arize Phoenix, Langfuse, or built-in log) without blocking the pipeline. `EvalSampler` runs alongside as a TimedAction, ticking every 30 minutes and scoring sampled EXPORTED items.

## Interaction sequence

The sequence traces the happy path (J1 + J2 combined). `RouterAgent` is always the first agent call; `SummaryAgent` is conditional on a SUMMARISE decision. Span events accumulate on `WorkloadItemEntity` as the workflow steps complete; `TraceExporter` picks them up asynchronously. The `Note over E,B` block marks the non-blocking export — a backend outage here does not interrupt the workflow.

## State machine

Ten states. The interesting branches:

- After ROUTED, the routing category branches the flow: SUMMARISE proceeds to PROCESSING; ESCALATE jumps to ESCALATED terminal; SKIP jumps to SKIPPED terminal.
- PROCESSING transitions to EXPORTED when `TracesExported` is emitted at the end of the workflow.
- EXPORTED can transition to EVALUATED when `EvalSampler` scores the item. Both are terminal from a processing standpoint; EVALUATED simply carries additional quality metadata.
- A timeout in `routeStep` emits `ItemFailed` and terminates — this is the only failure path visible in the state machine.

## Entity model

`WorkloadItemEntity` is the source of truth; it emits nine distinct event types covering the full lifecycle including tracing and eval. `WorkloadQueue` is the upstream audit log — only `TraceCollector` subscribes to it. `TraceExporter` subscribes directly to `WorkloadItemEntity`'s `SpanEmitted` events, which is an intentional architectural separation: span export is infrastructure, not domain logic.

## Trace export architecture

The `TraceExporter` Consumer is the governance-risk equivalent of the inbox-watcher's PiiSanitizer — it sits at the infrastructure layer and captures observability data without requiring agent code to be aware of it. This means:

1. Adding a new agent to the pipeline does not require updating `TraceExporter` — it only needs to emit `SpanEmitted` events.
2. Switching from Arize Phoenix to Langfuse (or disabling external export entirely) is a single env-var change.
3. The built-in structured log is always active, giving a baseline audit trail even when no external backend is configured.
