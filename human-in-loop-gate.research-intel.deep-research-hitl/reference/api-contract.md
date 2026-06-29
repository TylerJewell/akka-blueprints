# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/research-request` | `{ "query": "string" }` | `{ "reportId": "uuid" }` | ResearchEndpoint → ResearchWorkflow |
| POST | `/api/reports/{reportId}/approve` | `{ "reviewedBy": "string", "notes": "string" }` | `200` \| `404` | ResearchEndpoint → ResearchEntity |
| POST | `/api/reports/{reportId}/reject` | `{ "reviewedBy": "string", "notes": "string" }` | `200` \| `404` | ResearchEndpoint → ResearchEntity |
| GET | `/api/reports` | — | `{ "reports": [Report, ...] }` | ResearchEndpoint → ReportsView |
| GET | `/api/reports/{reportId}` | — | `Report` \| `404` | ResearchEndpoint → ResearchEntity |
| GET | `/api/reports/sse` | — | SSE stream of `Report` | ResearchEndpoint → ReportsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ResearchEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ResearchEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ResearchEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Report` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "query": "string or null",
  "status": "PLANNING | INVESTIGATING | SYNTHESISING | AWAITING_REVIEW | APPROVED | NEEDS_REVISION | DELIVERED",
  "planCreatedAt": "ISO-8601 or null",
  "subTopics": ["string", "..."] or null,
  "findingsSummaries": ["string", "..."] or null,
  "synthesisStartedAt": "ISO-8601 or null",
  "reportTitle": "string or null",
  "reportBody": "string or null",
  "sourcesUsed": ["url", "..."] or null,
  "awaitingReviewAt": "ISO-8601 or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNotes": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "revisionNotes": "string or null",
  "deliveredAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/reports/sse` streams the `ReportsView` rows. Each message:

```
event: report
data: { ...Report... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
