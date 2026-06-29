# Akka Sample: Workflow Observability (Arize/Langfuse)

A continuous background worker executes agent workflows, emits structured traces to Arize Phoenix and Langfuse, and surfaces a live observability dashboard showing span latency, token counts, and a periodic quality evaluation of agent outputs. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms (deployer-runtime-monitoring and periodic eval).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None** (built-in logging only by default; Arize Phoenix and Langfuse endpoints are optional and configured via env vars — the service works without them).

## Generate the system

```sh
cp -r ./continuous-monitor.governance-risk.workflow-observability  ~/my-projects/workflow-observability
cd ~/my-projects/workflow-observability
```

(Optional) Edit `SPEC.md` to point `TraceExporter` at a real Arize Phoenix or Langfuse endpoint, or keep the built-in in-memory trace store.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WorkloadPoller** — TimedAction firing every 20 s that drips simulated workload items into `WorkloadQueue`.
- **TraceCollector** — Consumer that intercepts `WorkloadItemReceived` events and opens a parent span.
- **RouterAgent** — Agent (typed) that classifies each workload item, selecting the right sub-agent.
- **SummaryAgent** — AutonomousAgent that processes the workload item and produces a structured result.
- **ObservabilityWorkflow** — Workflow per item: route → process → export traces → wait for eval.
- **WorkloadItemEntity** — EventSourcedEntity holding each item's lifecycle (queued → routed → processed → exported → evaluated).
- **TraceExporter** — Consumer that reads `TracingSpan` events and forwards them to Arize Phoenix, Langfuse, or the built-in trace log.
- **EvalSampler** — TimedAction running every 30 minutes; samples completed items, scores output quality.
- **ObservabilityView + ObservabilityEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — point `TraceExporter` at a real Arize Phoenix or Langfuse endpoint via env vars.
- `SPEC.md §5` — extend the `WorkloadItem` record with org-specific fields (`priority`, `tenantId`, `modelVersion`, etc.).
- `prompts/router-agent.md` — narrow routing rules to your specific workload taxonomy.
- `eval-matrix.yaml` — add regulation anchors for your jurisdiction's AI transparency requirements.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated workload item arrives → it is routed → processed → its spans appear in the trace log.
2. Spans include: item routing latency, LLM call duration, token counts, and processing result.
3. The EvalSampler scores at least one completed item within 30 minutes of start.
4. The App UI tab shows a live trace feed with span details and the eval score when populated.

## License

Apache 2.0.
