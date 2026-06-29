# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/locations` | `{ "address": "string", "city": "string", "region": "string" }` | `{ "siteId": "uuid" }` | `LocationEndpoint` → `SiteQueue` |
| GET | `/api/locations` | — | `{ "sites": [CandidateSiteRow, ...] }` | `LocationEndpoint` → `LocationView` |
| GET | `/api/locations?status=RECOMMENDED` | — | filtered list (client-side filter) | `LocationEndpoint` |
| GET | `/api/locations/{id}` | — | `CandidateSiteRow` or 404 | `LocationEndpoint` |
| GET | `/api/locations/sse` | — | `text/event-stream` of `CandidateSiteRow` | `LocationEndpoint` → `LocationView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `LocationEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `LocationEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `LocationEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/locations` request:

```json
{ "address": "742 Evergreen Terrace", "city": "Springfield", "region": "IL" }
```

`CandidateSiteRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "siteId": "uuid",
  "address": "string",
  "city": "string",
  "region": "string",
  "status": "SCORING | IN_PROGRESS | RECOMMENDED | NOT_RECOMMENDED | DEGRADED | BLOCKED",
  "marketAssessment": {
    "trafficScore": 0.78,
    "competitorDensity": 0.45,
    "tradeAreaClassification": "suburban-power-strip",
    "keyInsights": ["..."],
    "assessedAt": "ISO-8601"
  },
  "demographicAssessment": {
    "populationScore": 0.65,
    "incomeAlignmentScore": 0.72,
    "consumerProfile": "suburban families — dual income",
    "keyInsights": ["..."],
    "assessedAt": "ISO-8601"
  },
  "recommendation": {
    "summary": "string (80–150 words)",
    "score": 0.73,
    "verdict": "RECOMMENDED",
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

`GET /api/locations/sse` emits one event per site change:

```
event: site
data: { ...CandidateSiteRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `siteId`. No polling.
