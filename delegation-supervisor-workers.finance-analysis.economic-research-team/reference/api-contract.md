# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/analysis` | `{ "question": "string" }` | `{ "reportId": "uuid" }` | `AnalysisEndpoint` → `RequestQueue` |
| GET | `/api/analysis` | — | `{ "reports": [AnalysisReportRow, ...] }` | `AnalysisEndpoint` → `AnalysisView` |
| GET | `/api/analysis?status=PUBLISHED` | — | filtered list (client-side filter) | `AnalysisEndpoint` |
| GET | `/api/analysis/{id}` | — | `AnalysisReportRow` or 404 | `AnalysisEndpoint` |
| GET | `/api/analysis/sse` | — | `text/event-stream` of `AnalysisReportRow` | `AnalysisEndpoint` → `AnalysisView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `AnalysisEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `AnalysisEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `AnalysisEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/analysis` request:

```json
{ "question": "How does rising US inflation affect emerging-market bond yields?" }
```

`AnalysisReportRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reportId": "uuid",
  "question": "string",
  "status": "FRAMING | IN_PROGRESS | PUBLISHED | DEGRADED | BLOCKED",
  "indicators": {
    "indicators": [
      { "name": "CPI Year-over-Year", "value": "3.2", "unit": "percent", "source": "BLS 2024" }
    ],
    "collectedAt": "ISO-8601"
  },
  "interpretation": {
    "thesis": "string",
    "implications": ["..."],
    "interpretedAt": "ISO-8601"
  },
  "synthesised": {
    "summary": "string",
    "disclaimerVerdict": "ok",
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

`GET /api/analysis/sse` emits one event per report change:

```
event: report
data: { ...AnalysisReportRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `reportId`. No polling.
