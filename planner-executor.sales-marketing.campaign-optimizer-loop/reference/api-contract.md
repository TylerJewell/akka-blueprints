# API contract — campaign-optimizer-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/campaigns` | `{ "goal": String, "targetAudience": String, "channels": String, "requestedBy"?: String }` | `202 { "campaignId": String }` | `CampaignEndpoint` → `RequestQueue` |
| `GET` | `/api/campaigns` | — | `200 [ CampaignRow... ]` | `CampaignEndpoint` ← `CampaignView` |
| `GET` | `/api/campaigns/{id}` | — | `200 Campaign` / `404` | `CampaignEndpoint` ← `CampaignEntity` |
| `GET` | `/api/campaigns/sse` | — | `text/event-stream` (one event per campaign change) | `CampaignEndpoint` ← `CampaignView` |
| `POST` | `/api/campaigns/{id}/approve` | `{ "decidedBy": String, "note"?: String }` | `200 { "approved": true, "decidedBy": String }` | `CampaignEndpoint` → `ApprovalEntity` |
| `POST` | `/api/campaigns/{id}/reject` | `{ "decidedBy": String, "note"?: String }` | `200 { "approved": false, "decidedBy": String }` | `CampaignEndpoint` → `ApprovalEntity` |
| `GET` | `/api/campaigns/{id}/approval` | — | `200 { "approved"?: boolean, "decidedBy"?: String, "note"?: String, "decidedAt"?: Instant }` | `CampaignEndpoint` ← `ApprovalEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Campaign (full form, returned by `GET /api/campaigns/{id}`)

```json
{
  "campaignId": "c-8a1…",
  "goal": "Run a product-launch email campaign targeting enterprise buyers in North America.",
  "status": "AWAITING_APPROVAL",
  "ledger": {
    "goals": ["drive trial sign-ups among enterprise buyers in North America"],
    "targetAudience": ["enterprise decision-makers", "North America", "company size > 500"],
    "channelPlan": [
      "Draft email subject line and body copy",
      "Select enterprise-NA audience segment",
      "Publish email campaign assets",
      "Analyze open rate and click-through after 48 h"
    ],
    "brandConstraints": ["no competitor mentions", "no superlative claims"],
    "currentDispatch": null
  },
  "runLedger": {
    "entries": [
      {
        "attempt": 1,
        "specialist": "COPY",
        "step": "Draft email subject line and body copy",
        "verdict": "OK",
        "output": "Subject: Your Q3 Enterprise Offer Awaits\n\nBody: Discover how ...",
        "metricsSnapshot": { "openRate": null, "clickRate": null, "conversionRate": null, "impressions": null },
        "blocker": null,
        "recordedAt": "2026-06-28T09:14:02Z"
      },
      {
        "attempt": 1,
        "specialist": "AUDIENCE",
        "step": "Select enterprise-NA audience segment",
        "verdict": "OK",
        "output": "{\"segmentName\":\"Enterprise-NA-500+\",\"estimatedReach\":4200,\"cohortTags\":[\"enterprise\",\"north-america\",\"decision-maker\"]}",
        "metricsSnapshot": { "openRate": null, "clickRate": null, "conversionRate": null, "impressions": null },
        "blocker": null,
        "recordedAt": "2026-06-28T09:14:11Z"
      }
    ]
  },
  "report": null,
  "failureReason": null,
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:13:55Z",
  "finishedAt": null
}
```

### Campaign (list form, returned by `GET /api/campaigns`)

The list form is `CampaignRow` — same fields as `Campaign`, but `runLedger.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full campaign by id on row expand.

### SSE event format

```
event: campaign-update
data: { "campaignId": "c-8a1…", "status": "AWAITING_APPROVAL", ... }
```

One event per state transition. Clients reconcile by `campaignId`. The SSE channel also emits `event: approval-update` whenever an approval decision is recorded:

```
event: approval-update
data: { "campaignId": "c-8a1…", "approved": true, "decidedBy": "maria@example.com", "decidedAt": "2026-06-28T09:18:44Z" }
```
