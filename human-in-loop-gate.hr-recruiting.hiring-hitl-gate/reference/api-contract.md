# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/hiring-request` | `{ "candidateName": "string", "role": "string", "applicationSummary": "string" }` | `{ "applicationId": "uuid" }` | HiringEndpoint → HiringWorkflow |
| POST | `/api/applications/{applicationId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | HiringEndpoint → ApplicationEntity |
| POST | `/api/applications/{applicationId}/decline` | `{ "declinedBy": "string", "reason": "string" }` | `200` \| `404` | HiringEndpoint → ApplicationEntity |
| GET | `/api/applications` | — | `{ "applications": [Application, ...] }` | HiringEndpoint → ApplicationsView |
| GET | `/api/applications/{applicationId}` | — | `Application` \| `404` | HiringEndpoint → ApplicationEntity |
| GET | `/api/applications/sse` | — | SSE stream of `Application` | HiringEndpoint → ApplicationsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | HiringEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | HiringEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | HiringEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Application` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "candidateName": "string or null",
  "role": "string or null",
  "status": "PROPOSED | APPROVED | DECLINED | DECIDED",
  "submittedAt": "ISO-8601 or null",
  "proposedAt": "ISO-8601 or null",
  "hiringRecommendation": "string or null",
  "hiringRationale": "string or null",
  "meetingSubject": "string or null",
  "meetingBody": "string or null",
  "meetingSuggestedSlots": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "declinedAt": "ISO-8601 or null",
  "declinedBy": "string or null",
  "declineReason": "string or null",
  "decidedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/applications/sse` streams the `ApplicationsView` rows. Each message:

```
event: application
data: { ...Application... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
