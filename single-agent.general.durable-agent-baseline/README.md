# Akka Sample: Durable Agent Baseline

A single long-running agent accepts a multi-step work order, persists its own progress across JVM restarts, and resumes exactly where it left off. Demonstrates durable execution without external orchestration — the agent owns its state, the workflow owns its position, and the entity is the audit record.

Governance wires a runtime-monitoring hook that surfaces stall detection and resource consumption alongside the agent run, and a periodic performance evaluator that scores latency and completion-rate across restarts.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the work-order corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.general.durable-agent-baseline  ~/my-projects/durable-agent-baseline
cd ~/my-projects/durable-agent-baseline
```

(Optional) Edit `SPEC.md` to change the seeded work-order catalogue (e.g., replace the generic pipeline-step orders with domain-specific multi-stage tasks for your use case).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WorkOrderAgent** — an AutonomousAgent that accepts a multi-step work order as a task, processes each step in sequence, and emits a typed `WorkOrderResult`.
- **WorkOrderWorkflow** — orchestrates the per-work-order lifecycle: initiate → run agent → score → done. Survives JVM restarts without losing its position.
- **WorkOrderEntity** — an EventSourcedEntity holding the per-work-order audit trail (each step's outcome, timestamps, retries).
- **RuntimeMonitor** — a Consumer that subscribes to entity events, tracks stall windows and resource signals, and emits `StallDetected` or `ResourceAlert` when thresholds trip.
- **WorkOrderView + WorkOrderEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the seeded work-order catalogue with your own (the JSONL file under `src/main/resources/sample-events/work-orders.jsonl` after generation).
- `SPEC.md §5` — extend `WorkOrderResult` with domain-specific fields (e.g., `costEstimate`, `assignedTo`, `priorityClass`).
- `prompts/work-order-agent.md` — narrow the agent's role for your domain (a finance deployer might constrain it to GL-reconciliation steps; a DevOps deployer to pipeline-stage execution).
- `eval-matrix.yaml` — replace the mock runtime-monitoring signals with real CPU/memory metrics from your infrastructure by naming the source under the monitor mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a work order → the agent runs all steps → a completed result appears in the UI.
2. The service is restarted mid-execution → the workflow resumes from the last completed step, not from the beginning.
3. The runtime monitor detects a stall when the agent takes longer than its step timeout → a `StallDetected` event is recorded and visible in the UI.
4. The periodic evaluator scores latency and completion-rate across all work orders and shows the aggregate on the Eval Matrix tab.

## License

Apache 2.0.
