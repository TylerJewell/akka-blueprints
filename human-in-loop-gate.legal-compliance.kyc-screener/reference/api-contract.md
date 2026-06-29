# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/screening-request` | `{ "entityId": "string", "documents": ["string", ...] }` | `{ "caseId": "uuid" }` | ScreeningEndpoint → ScreeningWorkflow |
| POST | `/api/cases/{caseId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | ScreeningEndpoint → CaseEntity |
| POST | `/api/cases/{caseId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | ScreeningEndpoint → CaseEntity |
| GET | `/api/cases` | — | `{ "cases": [Case, ...] }` | ScreeningEndpoint → CasesView |
| GET | `/api/cases/{caseId}` | — | `Case` \| `404` | ScreeningEndpoint → CaseEntity |
| GET | `/api/cases/sse` | — | SSE stream of `Case` | ScreeningEndpoint → CasesView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ScreeningEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ScreeningEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ScreeningEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Case` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "entityId": "string or null",
  "status": "SCREENED | OFFICER_APPROVED | REJECTED | CLOSED",
  "screenedAt": "ISO-8601 or null",
  "findings": "string or null",
  "recommendation": "PASS | REFER | BLOCK | null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "closedAt": "ISO-8601 or null",
  "disposition": "APPROVED | null"
}
```

## SSE event format

`GET /api/cases/sse` streams the `CasesView` rows. Each message:

```
event: case
data: { ...Case... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
