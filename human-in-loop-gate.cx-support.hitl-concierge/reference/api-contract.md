# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/cases` | `{ "customerRequest": "string" }` | `{ "caseId": "uuid" }` | ConciergeEndpoint → ConciergeWorkflow |
| POST | `/api/cases/{caseId}/approve` | `{ "approvedBy": "string", "comment": "string" }` | `200` \| `404` | ConciergeEndpoint → CaseEntity |
| POST | `/api/cases/{caseId}/decline` | `{ "declinedBy": "string", "reason": "string" }` | `200` \| `404` | ConciergeEndpoint → CaseEntity |
| GET | `/api/cases` | — | `{ "cases": [Case, ...] }` | ConciergeEndpoint → CasesView |
| GET | `/api/cases/{caseId}` | — | `Case` \| `404` | ConciergeEndpoint → CaseEntity |
| GET | `/api/cases/sse` | — | SSE stream of `Case` | ConciergeEndpoint → CasesView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ConciergeEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ConciergeEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ConciergeEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Case` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "customerRequest": "string or null",
  "status": "TRIAGED | APPROVED | DECLINED | FULFILLED",
  "triagedAt": "ISO-8601 or null",
  "triageSummary": "string or null",
  "proposedResolution": "string or null",
  "urgency": "LOW | MEDIUM | HIGH | null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "specialistComment": "string or null",
  "declinedAt": "ISO-8601 or null",
  "declinedBy": "string or null",
  "declineReason": "string or null",
  "fulfilledAt": "ISO-8601 or null",
  "confirmationId": "string or null"
}
```

## SSE event format

`GET /api/cases/sse` streams the `CasesView` rows. Each message:

```
event: case
data: { ...Case... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
