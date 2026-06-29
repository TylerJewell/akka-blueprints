# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/reports` | `{ "topic": "string" }` | `{ "reportId": "uuid" }` | `ReportEndpoint` -> `TopicQueue` |
| GET | `/api/reports` | -- | `{ "reports": [ReportRow, ...] }` | `ReportEndpoint` -> `ReportView` |
| GET | `/api/reports?status=ASSEMBLED` | -- | filtered list (client-side filter) | `ReportEndpoint` |
| GET | `/api/reports/{id}` | -- | `ReportRow` or 404 | `ReportEndpoint` |
| GET | `/api/reports/sse` | -- | `text/event-stream` of `ReportRow` | `ReportEndpoint` -> `ReportView` |
| GET | `/api/metadata/readme` | -- | `text/markdown` | `ReportEndpoint` |
| GET | `/api/metadata/risk-survey` | -- | `text/yaml` | `ReportEndpoint` |
| GET | `/api/metadata/eval-matrix` | -- | `text/yaml` | `ReportEndpoint` |
| GET | `/` | -- | 302 -> `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | -- | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/reports` request:

```json
{ "topic": "How is AI reshaping academic peer review?" }
```

`ReportRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reportId": "uuid",
  "topic": "string",
  "status": "PLANNING | IN_PROGRESS | ASSEMBLED | DEGRADED | BLOCKED",
  "sources": {
    "sources": [
      { "title": "...", "reference": "...", "excerpt": "..." }
    ],
    "gatheredAt": "ISO-8601"
  },
  "draft": {
    "narrativeText": "...",
    "keyPoints": ["..."],
    "draftedAt": "ISO-8601"
  },
  "assembled": {
    "headline": "...",
    "body": "...",
    "guardrailVerdict": "ok",
    "assembledAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "qualityScore": "1-5 or null",
  "qualityRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/reports/sse` emits one event per report change:

```
event: report
data: { ...ReportRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `reportId`. No polling.
