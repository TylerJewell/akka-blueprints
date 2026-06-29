# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/reports` | `{ "ticker": "string", "question": "string" }` | `{ "reportId": "uuid" }` | `ReportEndpoint` → `RequestQueue` |
| GET | `/api/reports` | — | `{ "reports": [ResearchReportRow, ...] }` | `ReportEndpoint` → `ReportView` |
| GET | `/api/reports?status=PUBLISHED` | — | filtered list (client-side filter) | `ReportEndpoint` |
| GET | `/api/reports/{id}` | — | `ResearchReportRow` or 404 | `ReportEndpoint` |
| GET | `/api/reports/sse` | — | `text/event-stream` of `ResearchReportRow` | `ReportEndpoint` → `ReportView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ReportEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ReportEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ReportEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/reports` request:

```json
{ "ticker": "NVDA", "question": "What is NVIDIA's revenue growth trajectory and near-term sentiment?" }
```

`ResearchReportRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reportId": "uuid",
  "ticker": "NVDA",
  "question": "string",
  "status": "PLANNING | IN_PROGRESS | PUBLISHED | DEGRADED | BLOCKED",
  "fundamentals": {
    "facts": [
      { "metric": "EPS", "value": "16.84", "period": "Q3 FY2025", "source": "NVIDIA 10-Q 2024-11" }
    ],
    "gatheredAt": "ISO-8601"
  },
  "sentiment": {
    "overallSentiment": "bullish",
    "signals": [
      { "headline": "Data-center demand exceeds supply", "direction": "positive", "source": "unsourced — knowledge" }
    ],
    "gatheredAt": "ISO-8601"
  },
  "synthesised": {
    "summary": "string (80–150 words)",
    "sanitizerVerdict": "ok",
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

`GET /api/reports/sse` emits one event per report change:

```
event: report
data: { ...ResearchReportRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `reportId`. No polling.
