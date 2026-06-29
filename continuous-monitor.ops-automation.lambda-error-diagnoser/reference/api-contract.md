# API contract — lambda-error-diagnoser

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/incidents` | — | `200 [ Incident... ]` (sorted by severity desc, then newest-first) | `IncidentEndpoint` ← `IncidentView` |
| `GET` | `/api/incidents/{id}` | — | `200 Incident` / `404` | `IncidentEndpoint` ← `IncidentView` |
| `POST` | `/api/incidents/{id}/dismiss` | `{ "dismissedBy": String, "reason": String }` | `200 Incident` (status now DISMISSED) | `IncidentEndpoint` → `IncidentEntity` |
| `POST` | `/api/incidents/{id}/reopen` | `{ "reopenedBy": String }` | `200 Incident` (status now REOPENED) | `IncidentEndpoint` → `IncidentEntity` |
| `GET` | `/api/incidents/sse` | — | `text/event-stream` | `IncidentEndpoint` ← `IncidentView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### Incident

```json
{
  "incidentId": "inc-7a3f…",
  "raw": {
    "logId": "log-2bf…",
    "functionName": "order-processor",
    "functionArn": "arn:aws:lambda:us-east-1:123456789:function:order-processor",
    "errorType": "Runtime.ExitError",
    "errorMessage": "Communications link failure to rds.amazonaws.com",
    "stackTraceExcerpt": "com.example.db.ConnectionPool.acquire(ConnectionPool.java:88)\n...",
    "coldStart": false,
    "awsRegion": "us-east-1",
    "detectedAt": "2026-06-28T09:12:00Z"
  },
  "normalised": {
    "functionName": "order-processor",
    "errorCategory": "dependency-failure",
    "errorMessage": "Communications link failure to rds.amazonaws.com",
    "stackTraceExcerpt": "com.example.db.ConnectionPool.acquire(ConnectionPool.java:88)\n...",
    "coldStart": false,
    "severity": "HIGH"
  },
  "diagnosis": {
    "rootCause": "Connection pool exhaustion from a downstream RDS instance under high concurrency.",
    "fixSuggestion": "Enable RDS Proxy and set the max_connections parameter to match the function's reserved concurrency limit.",
    "confidence": "high",
    "reasoning": "errorMessage names the RDS host and stack trace shows the pool acquire path."
  },
  "eval": {
    "score": 5,
    "rationale": "Root cause is specific, fix is immediately actionable, and both are consistent with the error message."
  },
  "status": "RESOLVED",
  "dismissReason": null,
  "createdAt": "2026-06-28T09:12:00Z",
  "finishedAt": "2026-06-28T09:12:18Z"
}
```

### SSE event format

```
event: incident-update
data: { "incidentId": "...", "status": "RESOLVED", "eval": { "score": 5, "rationale": "..." }, ... }
```

One event per state transition. Clients reconcile by `incidentId`.

## Query parameters

### `GET /api/incidents`

| Parameter | Values | Default |
|---|---|---|
| `status` | `DETECTED`, `NORMALISED`, `DIAGNOSED`, `EVALUATED`, `RESOLVED`, `DISMISSED`, `REOPENED` | all |
| `severity` | `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` | all |

Both parameters filter on the server side before returning results.
