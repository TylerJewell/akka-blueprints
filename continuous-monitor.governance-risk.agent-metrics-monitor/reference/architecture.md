# Architecture — agent-metrics-monitor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`MetricsPoller` is the heartbeat — a TimedAction ticking every 60 s that writes `BatchReceived` events into `RawMetricsQueue` (event-sourced for audit). A `MetricsIngestor` Consumer subscribes to that queue, normalises each row into a typed `AgentMetricSample`, and emits `MetricSampleRecorded` on the per-agent `AgentMetricsEntity`. It also starts a `MetricsScanWorkflow` per batch. The workflow orchestrates: detect anomalies per agent → narrate a batch summary → record results back on the entities.

`EvalRunner` runs alongside as a second TimedAction, ticking every 30 minutes and auditing un-scored health summaries via `SummaryEvalAgent`.

## Interaction sequence

The sequence traces the path from a polled batch through to a health card update in the UI. There are two natural pauses: (1) the `MetricsIngestor` Consumer's subscription lag (sub-second in local dev), and (2) the workflow's `detectStep` fan-out, which calls `AnomalyDetectorAgent` once per agent before proceeding to `narrateStep`. All three agent calls are synchronous typed agents — they return a shaped result immediately, with no tool-use loop.

## State machine

`AgentMetricsEntity` is not a linear lifecycle machine. Its health state can transition freely in any direction — HEALTHY → CRITICAL → HEALTHY — as successive batches arrive. There are no terminal states; the entity accumulates metric history indefinitely (capped at 10 samples in memory; full audit in `RawMetricsQueue`).

The three non-status events (`MetricSampleRecorded`, `SummaryRecorded`, `EvalScored`) do not change `currentStatus` — they enrich the entity's data fields. Only `AnomalyDetected` updates `currentStatus`.

## Entity model

`AgentMetricsEntity` is the source of truth for per-agent health; it emits four event types. `RawMetricsQueue` is the upstream audit log — only `MetricsIngestor` subscribes to it. `MetricsView` projects from `AgentMetricsEntity` events and serves the list endpoint; the detail endpoint reads directly from the entity for the full `recentSamples` list and `latestSummary`.

## Eval governance flow

Every health summary the system surfaces on the dashboard has an independent quality check:

1. `SummaryNarratorAgent` produces the narrative from the batch's `AnomalyResult` set.
2. `EvalRunner` independently scores that narrative against the same `AnomalyResult` data.
3. Score and rationale are written back to the entity and rendered as a chip alongside the summary.

A falling average eval score over time signals that `SummaryNarratorAgent`'s output is drifting from the underlying data — either because the prompt needs adjustment or because the model's behaviour has changed. The eval signal is non-blocking: the dashboard remains operational while scores are pending.
