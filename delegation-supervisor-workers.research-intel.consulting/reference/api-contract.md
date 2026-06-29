# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/engagements` | `{ "brief": "string" }` | `{ "engagementId": "uuid" }` | `EngagementEndpoint` → `InboundRequestQueue` |
| POST | `/api/engagements/{id}/compliance` | `ComplianceReview` | `200` \| `404` | `EngagementEndpoint` → `EngagementEntity` |
| GET | `/api/engagements` | — | `{ "engagements": [Engagement, ...] }` | `EngagementEndpoint` → `EngagementsView` |
| GET | `/api/engagements/{id}` | — | `Engagement` \| `404` | `EngagementEndpoint` → `EngagementEntity` |
| GET | `/api/engagements/sse` | — | SSE stream of `Engagement` | `EngagementEndpoint` → `EngagementsView` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `EngagementEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `EngagementEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `EngagementEndpoint` |
| GET | `/` | — | `302 /app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

`GET /api/engagements` may carry an optional `?status=...` query; the endpoint filters the `getAllEngagements` result client-side because the view query cannot index the enum column (Lesson 2).

## Payload shapes

```json
// POST /api/engagements
{ "brief": "Size the EU market for industrial heat pumps." }

// POST /api/engagements/{id}/compliance  (ComplianceReview)
{ "reviewer": "string", "verdict": "PASS | FLAG", "notes": "string" }
```

## Engagement JSON form

Lifecycle fields are `Optional<T>` in Java; on the wire they are the raw value or `null`.

```json
{
  "id": "uuid",
  "brief": "string or null",
  "status": "RECEIVED | ROUTED | RESEARCHING | CONSULTING | DELIVERED | COMPLIANCE_REVIEWED | FLAGGED",
  "route": "DELEGATE | HANDOFF | null",
  "complexityScore": "number or null",
  "routingRationale": "string or null",
  "routedAt": "ISO-8601 or null",
  "assignedTo": "junior | senior | null",
  "deliverableTitle": "string or null",
  "deliverableContent": "string or null",
  "deliveredAt": "ISO-8601 or null",
  "evalScore": "number or null",
  "evalNotes": "string or null",
  "complianceReviewedAt": "ISO-8601 or null",
  "complianceReviewer": "string or null",
  "complianceVerdict": "PASS | FLAG | null",
  "complianceNotes": "string or null"
}
```

## SSE event format

`GET /api/engagements/sse` emits one `data:` line per engagement change, each carrying the full `Engagement` JSON above:

```
event: engagement
data: { "id": "…", "status": "DELIVERED", ... }

```

The browser subscribes with `EventSource('/api/engagements/sse')` and upserts each engagement into the live list by `id`.
