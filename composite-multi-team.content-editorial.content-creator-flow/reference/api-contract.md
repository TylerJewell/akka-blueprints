# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/campaigns` | `{ "topic": "string" }` | `{ "campaignId": "uuid" }` | ContentEndpoint |
| GET | `/api/campaigns` | — | `{ "campaigns": [Campaign, ...] }` | ContentEndpoint |
| GET | `/api/campaigns/{id}` | — | `Campaign` \| 404 | ContentEndpoint |
| GET | `/api/campaigns/sse` | — | SSE stream of `Campaign` | ContentEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ContentEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ContentEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ContentEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static resource | AppEndpoint |

`GET /api/campaigns` accepts an optional `?status=` query; the endpoint filters client-side over `CampaignsView.getAllCampaigns` because the view cannot index the enum column (Lesson 2).

## Campaign JSON

Lifecycle fields are nullable — `Optional<T>` in Java, raw value or `null` on the wire.

```json
{
  "id": "uuid",
  "topic": "string or null",
  "status": "RECEIVED | RESEARCHING | DRAFTING | REVIEWING | EVALUATING | COMPLETED | BLOCKED",
  "researchReport": "string or null",
  "blogPost": "string or null",
  "linkedInPost": "string or null",
  "brandVerdict": "PASS | BLOCK or null",
  "brandNotes": "string or null",
  "qualityScore": "number or null",
  "qualityNotes": "string or null",
  "createdAt": "ISO-8601 or null",
  "researchedAt": "ISO-8601 or null",
  "draftedAt": "ISO-8601 or null",
  "reviewedAt": "ISO-8601 or null",
  "evaluatedAt": "ISO-8601 or null",
  "completedAt": "ISO-8601 or null",
  "blockedAt": "ISO-8601 or null",
  "blockReason": "string or null"
}
```

While a campaign is `BLOCKED`, `researchReport`, `blogPost`, and `linkedInPost` are withheld from the public list and SSE projections; `blockReason` is present.

## SSE event format

`GET /api/campaigns/sse` emits one event per `CampaignsView` update:

```
event: campaign
data: { ...Campaign JSON... }
```

The UI applies each event by `id`, replacing the matching row.
