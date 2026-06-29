# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/analyze` | `{ "companyName": "...", "domain": "..." }` (domain optional) | `{ "prospectId": "uuid" }` | `ProspectEndpoint` → `RequestQueueEntity` |
| GET | `/api/prospects` | — | `{ "prospects": [Prospect, ...] }` | `ProspectEndpoint` → `ProspectsView` |
| GET | `/api/prospects/{id}` | — | `Prospect` or 404 | `ProspectEndpoint` → `ProspectsView` |
| GET | `/api/prospects/sse` | — | SSE stream of `Prospect` | `ProspectEndpoint` → `ProspectsView` |
| GET | `/tools/research?domain=...` | — | `{ "facts": {...} }` or 403 (off-list) | `ResearchToolEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ProspectEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ProspectEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ProspectEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`Prospect` (lifecycle fields are nullable — `Optional` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "companyName": "string",
  "domain": "string or null",
  "status": "RESEARCHING | ANALYZING | STRATEGIZING | COMPLETED | STALLED",
  "researchedAt": "ISO-8601 or null",
  "companyProfile": { "industry": "string", "summary": "string", "hqLocation": "string", "employeeEstimate": 0 },
  "analyzedAt": "ISO-8601 or null",
  "orgStructureSummary": "string or null",
  "decisionMakers": [
    { "name": "string", "title": "string", "department": "string", "contactHint": "masked string or null" }
  ],
  "strategizedAt": "ISO-8601 or null",
  "outreachStrategy": { "approach": "string", "talkingPoints": ["string"], "recommendedChannel": "string" },
  "completedAt": "ISO-8601 or null",
  "stalledAt": "ISO-8601 or null"
}
```

`companyProfile`, `outreachStrategy` are `null` until their phase completes. `decisionMakers` is `[]` until analysis runs. Every `contactHint` shown has already passed the S1 sanitizer.

## SSE event format

`GET /api/prospects/sse` emits one event per `ProspectEntity` change:

```
event: prospect
data: { ...Prospect JSON as above... }
```

Clients reconcile by `id`, replacing the prior copy of each prospect.
