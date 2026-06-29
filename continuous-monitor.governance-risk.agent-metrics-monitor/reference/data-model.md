# Data model — agent-metrics-monitor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `AgentMetricSample` | `agentId` | `String` | no | Unique agent identifier. |
| | `agentName` | `String` | no | Human-readable name (e.g. "fraud-screener"). |
| | `windowStart` | `Instant` | no | Start of the measurement window. |
| | `windowEnd` | `Instant` | no | End of the measurement window. |
| | `invocationCount` | `int` | no | Total invocations in window. |
| | `errorRate` | `double` | no | Fraction 0.0–1.0. |
| | `p50LatencyMs` | `double` | no | Median latency in milliseconds. |
| | `p99LatencyMs` | `double` | no | 99th-percentile latency in milliseconds. |
| | `toolCallCount` | `int` | no | Total tool calls made in window. |
| | `batchId` | `String` | no | The batch this sample arrived in. |
| `AnomalyResult` | `agentId` | `String` | no | Matches `AgentMetricSample.agentId`. |
| | `status` | `HealthStatus` | no | One of `HEALTHY`, `DEGRADED`, `CRITICAL`. |
| | `signals` | `List<String>` | no | Threshold-breach descriptions. Empty for HEALTHY. |
| | `verdict` | `String` | no | One-sentence classification rationale. |
| `HealthSummary` | `batchId` | `String` | no | The batch this summary covers. |
| | `narrativeText` | `String` | no | 2–4 sentence health narrative. |
| | `agentCount` | `int` | no | Total agents in the batch. |
| | `criticalCount` | `int` | no | Count of CRITICAL agents. |
| | `degradedCount` | `int` | no | Count of DEGRADED agents. |
| | `producedAt` | `Instant` | no | When `SummaryNarratorAgent` finished. |
| `EvalResult` | `score` | `int` | no | 1–5 weighted average. |
| | `rationale` | `String` | no | One sentence naming the dominant scoring signal. |
| `AgentHealthState` (entity state) | `agentId` | `String` | no | — |
| | `agentName` | `String` | no | — |
| | `recentSamples` | `List<AgentMetricSample>` | no | Last 10; oldest dropped when capped. |
| | `latestAnomaly` | `Optional<AnomalyResult>` | yes | Populated after `AnomalyDetected`. |
| | `latestSummary` | `Optional<HealthSummary>` | yes | Populated after `SummaryRecorded`. |
| | `latestEvalScore` | `Optional<Integer>` | yes | Populated after `EvalScored`. |
| | `latestEvalRationale` | `Optional<String>` | yes | Populated after `EvalScored`. |
| | `currentStatus` | `HealthStatus` | no | Updated by every `AnomalyDetected` event. |
| | `firstSeenAt` | `Instant` | no | Timestamp of first `MetricSampleRecorded`. |
| | `lastUpdatedAt` | `Instant` | no | Timestamp of most recent event. |
| `RawMetricsBatch` | `batchId` | `String` | no | Unique id for the polled batch. |
| | `rows` | `List<AgentMetricSample>` | no | All rows in the batch. |
| | `receivedAt` | `Instant` | no | When `MetricsPoller` fetched the batch. |

## Enums

`HealthStatus`: `HEALTHY`, `DEGRADED`, `CRITICAL`.

## Events (`AgentMetricsEntity`)

| Event | Payload | Effect |
|---|---|---|
| `MetricSampleRecorded` | `AgentMetricSample` | Appends to `recentSamples` (cap 10); sets `agentName`, `firstSeenAt`, `lastUpdatedAt`. |
| `AnomalyDetected` | `AnomalyResult` | Updates `latestAnomaly` and `currentStatus`; sets `lastUpdatedAt`. |
| `SummaryRecorded` | `HealthSummary` | Updates `latestSummary`; clears `latestEvalScore` and `latestEvalRationale` (new summary resets eval pending state); sets `lastUpdatedAt`. |
| `EvalScored` | `score: int, rationale: String` | Populates `latestEvalScore` and `latestEvalRationale`; sets `lastUpdatedAt`. |

## Events (`RawMetricsQueue`)

| Event | Payload |
|---|---|
| `BatchReceived` | `RawMetricsBatch` (full raw batch; audit log for `MetricsIngestor` subscription) |

## View row

`AgentHealthRow` mirrors `AgentHealthState` minus the `recentSamples` list and the full `latestSummary` body. Fields included: `agentId`, `agentName`, `currentStatus`, `latestAnomalySignals` (from `latestAnomaly.signals`), `latestAnomalyVerdict` (from `latestAnomaly.verdict`), `latestSummaryExcerpt` (first sentence of `latestSummary.narrativeText`), `latestEvalScore`, `latestEvalRationale`, `lastUpdatedAt`. The full detail is fetched on demand via `GET /api/metrics/agents/{agentId}`.
