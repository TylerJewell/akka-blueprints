# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/campaigns` | `{ "objective": "string" }` | `{ "briefId": "uuid" }` | `CampaignEndpoint` → `ObjectiveQueue` |
| GET | `/api/campaigns` | — | `{ "briefs": [CampaignBriefRow, ...] }` | `CampaignEndpoint` → `CampaignView` |
| GET | `/api/campaigns?status=ASSEMBLED` | — | filtered list (client-side filter) | `CampaignEndpoint` |
| GET | `/api/campaigns/{id}` | — | `CampaignBriefRow` or 404 | `CampaignEndpoint` |
| GET | `/api/campaigns/sse` | — | `text/event-stream` of `CampaignBriefRow` | `CampaignEndpoint` → `CampaignView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `CampaignEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `CampaignEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `CampaignEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/campaigns` request:

```json
{ "objective": "Launch a developer-focused awareness campaign for a new API product in North America." }
```

`CampaignBriefRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "briefId": "uuid",
  "objective": "string",
  "status": "PLANNING | IN_PROGRESS | ASSEMBLED | DEGRADED | BLOCKED",
  "marketInsights": {
    "insights": [ { "headline": "...", "source": "...", "detail": "..." } ],
    "gatheredAt": "ISO-8601"
  },
  "audienceProfile": {
    "segments": [ { "name": "...", "description": "...", "characteristics": ["..."] } ],
    "primaryPersona": "...",
    "profiledAt": "ISO-8601"
  },
  "messagingGuide": {
    "coreMessage": "...",
    "supportingMessages": ["..."],
    "toneGuidance": "...",
    "createdAt": "ISO-8601"
  },
  "channelPlan": {
    "channels": [ { "channel": "...", "priority": "HIGH|MEDIUM|LOW", "justification": "..." } ],
    "rationale": "...",
    "createdAt": "ISO-8601"
  },
  "assembled": {
    "executiveSummary": "...",
    "complianceVerdict": "compliant",
    "assembledAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/campaigns/sse` emits one event per brief change:

```
event: brief
data: { ...CampaignBriefRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `briefId`. No polling.
