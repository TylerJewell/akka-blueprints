# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/score-request` | `{ "companyName": "string", "contactEmail": "string" }` | `{ "leadId": "uuid" }` | LeadScoringEndpoint → LeadScoringWorkflow |
| POST | `/api/leads/{leadId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | LeadScoringEndpoint → LeadEntity |
| POST | `/api/leads/{leadId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | LeadScoringEndpoint → LeadEntity |
| GET | `/api/leads` | — | `{ "leads": [Lead, ...] }` | LeadScoringEndpoint → LeadsView |
| GET | `/api/leads/{leadId}` | — | `Lead` \| `404` | LeadScoringEndpoint → LeadEntity |
| GET | `/api/leads/sse` | — | SSE stream of `Lead` | LeadScoringEndpoint → LeadsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | LeadScoringEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | LeadScoringEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | LeadScoringEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Lead` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "companyName": "string or null",
  "contactEmail": "string or null",
  "status": "SCORED | APPROVED | DISQUALIFIED | QUALIFIED",
  "scoredAt": "ISO-8601 or null",
  "score": "integer or null",
  "scoreRationale": "string or null",
  "scoreConfidence": "high | medium | low | null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "qualifiedAt": "ISO-8601 or null",
  "qualificationVerdict": "string or null",
  "qualificationNextSteps": "string or null"
}
```

## SSE event format

`GET /api/leads/sse` streams the `LeadsView` rows. Each message:

```
event: lead
data: { ...Lead... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
