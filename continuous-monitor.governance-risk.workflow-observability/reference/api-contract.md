# API contract — workflow-observability

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/traces` | — | `200 [ WorkloadItemRow... ]` (sorted newest-first; optional `?status=`, `?workloadType=`) | `ObservabilityEndpoint` <- `ObservabilityView` |
| `GET` | `/api/traces/{id}` | — | `200 WorkloadItemRow` / `404` | `ObservabilityEndpoint` <- `ObservabilityView` |
| `GET` | `/api/traces/{id}/spans` | — | `200 [ TracingSpan... ]` / `404` | `ObservabilityEndpoint` <- `ObservabilityView` |
| `POST` | `/api/traces` | `{ "workloadType": String, "payload": String, "sourceTag": String }` | `202 { "itemId": String }` | `ObservabilityEndpoint` -> `WorkloadQueue` |
| `GET` | `/api/traces/sse` | — | `text/event-stream` | `ObservabilityEndpoint` <- `ObservabilityView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 -> /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### WorkloadItemRow

```json
{
  "itemId": "wi-7a3f…",
  "item": {
    "itemId": "wi-7a3f…",
    "workloadType": "audit-trail-digest",
    "payload": "2026-06-28: 14 access events, 2 privilege escalations…",
    "sourceTag": "siem-export",
    "receivedAt": "2026-06-28T09:12:00Z"
  },
  "routing": {
    "category": "SUMMARISE",
    "confidence": "high",
    "rationale": "Routine audit digest, no anomaly indicators."
  },
  "result": {
    "summary": "The audit trail covers 14 access events on 2026-06-28, including 2 privilege escalations. No SIEM anomalies were flagged.",
    "keyFindings": "• 14 access events recorded\n• 2 privilege escalations (no SIEM alert)\n• No anomalies flagged",
    "outputTokens": 92,
    "processedAt": "2026-06-28T09:12:07Z"
  },
  "spans": [
    {
      "spanId": "sp-001",
      "parentSpanId": "sp-root",
      "operationName": "RouterAgent.route",
      "startedAt": "2026-06-28T09:12:01Z",
      "finishedAt": "2026-06-28T09:12:02Z",
      "durationMs": 980,
      "promptTokens": 210,
      "completionTokens": 35,
      "status": "OK",
      "errorMessage": null
    },
    {
      "spanId": "sp-002",
      "parentSpanId": "sp-root",
      "operationName": "SummaryAgent.process",
      "startedAt": "2026-06-28T09:12:02Z",
      "finishedAt": "2026-06-28T09:12:06Z",
      "durationMs": 4120,
      "promptTokens": 390,
      "completionTokens": 92,
      "status": "OK",
      "errorMessage": null
    }
  ],
  "eval": {
    "score": 5,
    "rationale": "Summary is factually grounded and key findings are precise.",
    "evaluatedAt": "2026-06-28T09:42:00Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T09:12:00Z",
  "completedAt": "2026-06-28T09:12:07Z"
}
```

### SSE event format

```
event: trace-update
data: { "itemId": "wi-7a3f…", "status": "EXPORTED", "spans": [...] }
```

One event per state transition. Clients reconcile by `itemId`. The `spans` array in SSE events is the full accumulated list at the time of the transition.
