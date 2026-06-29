# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/briefs` | `{ "brief": "string" }` | `{ "campaignId": "uuid" }` | CampaignEndpoint |
| GET | `/api/campaigns` | — (optional `?status=`) | `{ "campaigns": [Campaign, ...] }` | CampaignEndpoint |
| GET | `/api/campaigns/{id}` | — | `Campaign` or 404 | CampaignEndpoint |
| GET | `/api/campaigns/sse` | — | SSE stream of `Campaign` | CampaignEndpoint |
| GET | `/api/search` | — (`?q=terms`) | `{ "results": [SearchResult, ...] }` | WebSearchEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | CampaignEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | CampaignEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | CampaignEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

Status filtering on `/api/campaigns?status=` is applied client-side in the endpoint over the View's `getAllCampaigns` result (Lesson 2).

## Payload shapes

`Campaign` (lifecycle fields are nullable — `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "brief": "string or null",
  "status": "RECEIVED | RESEARCHED | STRATEGIZED | CONTENT_DRAFTED | ASSEMBLED | EVALUATED | FAILED",
  "receivedAt": "ISO-8601 or null",
  "findings": { "claims": ["string"], "sources": ["string"] },
  "researchedAt": "ISO-8601 or null",
  "strategy": { "positioning": "string", "channels": ["string"], "tactics": ["string"] },
  "strategizedAt": "ISO-8601 or null",
  "content": { "taglines": ["string"], "posts": ["string"] },
  "contentDraftedAt": "ISO-8601 or null",
  "planSummary": "string or null",
  "assembledAt": "ISO-8601 or null",
  "evalScore": "integer or null",
  "evalNotes": "string or null",
  "evaluatedAt": "ISO-8601 or null",
  "failureReason": "string or null",
  "failedAt": "ISO-8601 or null"
}
```

`SearchResult`:

```json
{ "title": "string", "snippet": "string", "source": "https://..." }
```

## SSE event format

`GET /api/campaigns/sse` emits one event per campaign change, body = the full `Campaign` JSON above:

```
event: campaign
data: { "id": "...", "status": "ASSEMBLED", ... }

```

The stream is backed by the View's `streamAllCampaigns` query via `serverSentEventsForView` — no polling.
