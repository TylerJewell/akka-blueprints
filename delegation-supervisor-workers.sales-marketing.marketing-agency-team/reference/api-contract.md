# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/campaigns` | `{ "campaignName": "string", "objective": "string" }` | `{ "planId": "uuid" }` | `CampaignEndpoint` → `RequestQueue` |
| GET | `/api/campaigns` | — | `{ "plans": [CampaignPlanRow, ...] }` | `CampaignEndpoint` → `CampaignView` |
| GET | `/api/campaigns?status=SYNTHESISED` | — | filtered list (client-side filter) | `CampaignEndpoint` |
| GET | `/api/campaigns/{id}` | — | `CampaignPlanRow` or 404 | `CampaignEndpoint` |
| GET | `/api/campaigns/sse` | — | `text/event-stream` of `CampaignPlanRow` | `CampaignEndpoint` → `CampaignView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `CampaignEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `CampaignEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `CampaignEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/campaigns` request:

```json
{ "campaignName": "Q3 Product Launch", "objective": "Drive 20% trial sign-up lift among SMB buyers" }
```

`CampaignPlanRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "planId": "uuid",
  "campaignName": "string",
  "objective": "string",
  "status": "PLANNING | IN_PROGRESS | SYNTHESISED | DEGRADED | BLOCKED",
  "launchBrief": {
    "launchItems": [
      { "task": "string", "priority": "HIGH | MEDIUM | LOW", "rationale": "string" }
    ],
    "channelRecommendation": "string",
    "preparedAt": "ISO-8601"
  },
  "strategyFramework": {
    "positioningStatement": "string",
    "messagingPillars": ["string"],
    "targetAudiences": ["string"],
    "preparedAt": "ISO-8601"
  },
  "synthesised": {
    "executiveSummary": "string",
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

`GET /api/campaigns/sse` emits one event per plan change:

```
event: plan
data: { ...CampaignPlanRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `planId`. No polling.
