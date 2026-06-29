# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/reports` | `{ "question": "string" }` | `{ "reportId": "uuid" }` | `ReportEndpoint` -> `QuestionQueue` |
| GET | `/api/reports` | -- | `{ "reports": [ResearchReportRow, ...] }` | `ReportEndpoint` -> `ReportView` |
| GET | `/api/reports?status=SYNTHESISED` | -- | filtered list (client-side filter) | `ReportEndpoint` |
| GET | `/api/reports/{id}` | -- | `ResearchReportRow` or 404 | `ReportEndpoint` |
| GET | `/api/reports/sse` | -- | `text/event-stream` of `ResearchReportRow` | `ReportEndpoint` -> `ReportView` |
| GET | `/api/metadata/readme` | -- | `text/markdown` | `ReportEndpoint` |
| GET | `/api/metadata/risk-survey` | -- | `text/yaml` | `ReportEndpoint` |
| GET | `/api/metadata/eval-matrix` | -- | `text/yaml` | `ReportEndpoint` |
| GET | `/` | -- | 302 -> `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | -- | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/reports` request:

```json
{ "question": "What are the primary obligations for general-purpose AI model providers under the EU AI Act?" }
```

`ResearchReportRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reportId": "uuid",
  "question": "string",
  "status": "PLANNING | IN_PROGRESS | SYNTHESISED | DEGRADED | BLOCKED",
  "plan": {
    "subqueries": [
      { "subqueryId": "sq-1", "queryText": "string" }
    ]
  },
  "subquerySummaries": [
    {
      "subqueryId": "sq-1",
      "queryText": "string",
      "summary": "string",
      "summarisedAt": "ISO-8601"
    }
  ],
  "synthesised": {
    "executiveSummary": "string",
    "citationList": [
      { "citationId": "c-1", "source": "string", "quote": "string" }
    ],
    "guardrailVerdict": "ok",
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
