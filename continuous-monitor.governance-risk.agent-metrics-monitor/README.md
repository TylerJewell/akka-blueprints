# Akka Sample: Agent Metrics Monitor

A continuous background worker queries BigQuery for agent execution telemetry, detects performance anomalies, generates natural-language summaries for human review, and surfaces a scored health signal on a live dashboard. Demonstrates the **continuous-monitor** coordination pattern wired with one governance mechanism (eval-periodic performance monitor).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None**. The blueprint ships a BigQuery simulator; no live GCP project is required.

## Generate the system

```sh
cp -r ./continuous-monitor.governance-risk.agent-metrics-monitor  ~/my-projects/agent-metrics-monitor
cd ~/my-projects/agent-metrics-monitor
```

(Optional) Edit `SPEC.md` to point `MetricsPoller` at a real BigQuery dataset or to keep the in-process simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MetricsPoller** — TimedAction firing every 60 s that pulls agent execution rows from a simulated BigQuery result set.
- **MetricsIngestor** — Consumer that normalises raw rows into typed `AgentMetricSample` records and fans them into `AgentMetricsEntity`.
- **AnomalyDetectorAgent** — Agent (typed) that inspects a window of recent samples and classifies each agent's health as `HEALTHY`, `DEGRADED`, or `CRITICAL`.
- **SummaryNarratorAgent** — Agent (typed) that produces a brief human-readable summary of the current health window.
- **AgentMetricsEntity** — EventSourcedEntity tracking per-agent metric history and current health state.
- **MetricsScanWorkflow** — Workflow that, for each polled batch, runs detect-anomalies → narrate → record results.
- **MetricsView + MetricsEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **EvalRunner** — TimedAction running every 30 minutes; audits the last N summaries for accuracy and grounding.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — wire `MetricsPoller` to a real BigQuery client by replacing the simulator source.
- `SPEC.md §5` — extend `AgentMetricSample` with org-specific fields (`region`, `costUsd`, `toolCallCount`).
- `prompts/anomaly-detector.md` — tighten the anomaly thresholds for your deployment's latency and error-rate baselines.
- `eval-matrix.yaml` — replace the LLM-judge eval with a deterministic threshold check if your compliance team requires auditability without an LLM in the eval path.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated batch of agent metrics arrives; each agent's health state appears in the UI within 60 s.
2. An agent with elevated error rate is classified `DEGRADED` or `CRITICAL`; the anomaly detail is visible.
3. The `SummaryNarratorAgent` produces a readable summary that appears on the Overview panel.
4. `EvalRunner` scores at least one summary within 30 minutes; the score appears in the UI.
5. Switching to a BigQuery simulator batch where all agents are healthy resets states to `HEALTHY` within the next poll cycle.

## License

Apache 2.0.
