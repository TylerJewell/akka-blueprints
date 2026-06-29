# API contract — agent-metrics-monitor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/metrics/agents` | — | `200 [ AgentHealthRow... ]` (sorted by severity: CRITICAL first) | `MetricsEndpoint` ← `MetricsView` |
| `GET` | `/api/metrics/agents?status=CRITICAL` | — | `200 [ AgentHealthRow... ]` filtered | `MetricsEndpoint` ← `MetricsView` |
| `GET` | `/api/metrics/agents/{agentId}` | — | `200 AgentHealthState` / `404` | `MetricsEndpoint` ← `AgentMetricsEntity` |
| `GET` | `/api/metrics/summaries` | — | `200 [ HealthSummary... ]` newest-first | `MetricsEndpoint` ← `MetricsView` |
| `GET` | `/api/metrics/sse` | — | `text/event-stream` | `MetricsEndpoint` ← `MetricsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### AgentHealthRow (list endpoint)

```json
{
  "agentId": "agent-002",
  "agentName": "fraud-screener",
  "currentStatus": "DEGRADED",
  "latestAnomalySignals": ["errorRate 0.14 > threshold 0.10"],
  "latestAnomalyVerdict": "Error rate above warning threshold.",
  "latestSummaryExcerpt": "Of 3 monitored agents, 1 is degraded and 2 are healthy.",
  "latestEvalScore": 4,
  "latestEvalRationale": "Counts and signals accurate; narrative is clear and concise.",
  "lastUpdatedAt": "2026-06-28T09:15:00Z"
}
```

### AgentHealthState (detail endpoint)

```json
{
  "agentId": "agent-002",
  "agentName": "fraud-screener",
  "recentSamples": [
    {
      "agentId": "agent-002",
      "agentName": "fraud-screener",
      "windowStart": "2026-06-28T09:00:00Z",
      "windowEnd": "2026-06-28T09:01:00Z",
      "invocationCount": 210,
      "errorRate": 0.14,
      "p50LatencyMs": 320.0,
      "p99LatencyMs": 3100.0,
      "toolCallCount": 840,
      "batchId": "batch-0042"
    }
  ],
  "latestAnomaly": {
    "agentId": "agent-002",
    "status": "DEGRADED",
    "signals": ["errorRate 0.14 > threshold 0.10"],
    "verdict": "Error rate above warning threshold; latency and throughput acceptable."
  },
  "latestSummary": {
    "batchId": "batch-0042",
    "narrativeText": "Of 3 monitored agents, 1 is degraded and 2 are healthy. The fraud-screener is degraded due to an elevated error rate (errorRate 0.14 > threshold 0.10); throughput and latency remain acceptable.",
    "agentCount": 3,
    "criticalCount": 0,
    "degradedCount": 1,
    "producedAt": "2026-06-28T09:15:02Z"
  },
  "latestEvalScore": 4,
  "latestEvalRationale": "Counts and signals accurate; narrative is clear and concise.",
  "currentStatus": "DEGRADED",
  "firstSeenAt": "2026-06-28T08:00:00Z",
  "lastUpdatedAt": "2026-06-28T09:15:00Z"
}
```

### SSE event format

```
event: agent-health-update
data: { "agentId": "agent-002", "currentStatus": "DEGRADED", "lastUpdatedAt": "..." }
```

One event per state change or data enrichment on `AgentMetricsEntity`. Clients reconcile by `agentId`.
