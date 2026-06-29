# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/analysis` | `{ "question": "string" }` | `{ "jobId": "uuid" }` | `AnalysisEndpoint` → `QueryQueue` |
| GET | `/api/analysis` | — | `{ "jobs": [AnalysisJobRow, ...] }` | `AnalysisEndpoint` → `AnalysisView` |
| GET | `/api/analysis?status=COMPLETE` | — | filtered list (client-side filter) | `AnalysisEndpoint` |
| GET | `/api/analysis/{id}` | — | `AnalysisJobRow` or 404 | `AnalysisEndpoint` |
| GET | `/api/analysis/sse` | — | `text/event-stream` of `AnalysisJobRow` | `AnalysisEndpoint` → `AnalysisView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `AnalysisEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `AnalysisEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `AnalysisEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/analysis` request:

```json
{ "question": "What were the top 5 revenue-generating product lines last quarter?" }
```

`AnalysisJobRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "jobId": "uuid",
  "question": "string",
  "status": "TRANSLATING | RUNNING | COMPLETE | DEGRADED | BLOCKED",
  "sqlPlan": {
    "queryId": "uuid",
    "sql": "SELECT product_line, SUM(amount_usd) AS total FROM revenue WHERE ...",
    "targetTable": "revenue",
    "safetyVerdict": "ok"
  },
  "dataSet": {
    "queryId": "uuid",
    "data": {
      "columns": ["product_line", "total"],
      "rows": [["Equities", "4200000"], ["Fixed Income", "3100000"]],
      "queriedAt": "ISO-8601"
    },
    "rowCount": 5,
    "retrievedAt": "ISO-8601"
  },
  "modelResult": {
    "modelType": "time_series_forecast",
    "metrics": [
      { "metricName": "mae", "value": 12340.50, "interpretation": "Mean absolute error of the quarterly forecast." }
    ],
    "modelledAt": "ISO-8601"
  },
  "report": {
    "summary": "string (80-150 words)",
    "synthesisedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/analysis/sse` emits one event per job change:

```
event: job
data: { ...AnalysisJobRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `jobId`. No polling.
