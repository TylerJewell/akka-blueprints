# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/advisory` | `{ "description": "string" }` | `{ "sessionId": "uuid" }` | `AdvisoryEndpoint` → `BriefQueue` |
| GET | `/api/advisory` | — | `{ "sessions": [AdvisorySessionRow, ...] }` | `AdvisoryEndpoint` → `AdvisoryView` |
| GET | `/api/advisory?status=ADVISED` | — | filtered list (client-side filter) | `AdvisoryEndpoint` |
| GET | `/api/advisory/{id}` | — | `AdvisorySessionRow` or 404 | `AdvisoryEndpoint` |
| GET | `/api/advisory/sse` | — | `text/event-stream` of `AdvisorySessionRow` | `AdvisoryEndpoint` → `AdvisoryView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `AdvisoryEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `AdvisoryEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `AdvisoryEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/advisory` request:

```json
{ "description": "B2B SaaS tool for engineering team hiring, targeting Series A startups, pre-revenue." }
```

`AdvisorySessionRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "sessionId": "uuid",
  "description": "string",
  "status": "SCOPING | IN_PROGRESS | ADVISED | DEGRADED",
  "marketSnapshot": {
    "tam": "string",
    "competitors": ["string"],
    "buyerSegments": [{ "name": "string", "painPoint": "string" }],
    "researchedAt": "ISO-8601"
  },
  "channelPlan": {
    "channels": [{ "channel": "string", "rationale": "string", "effort": "low|medium|high" }],
    "primaryChannel": "string",
    "analysedAt": "ISO-8601"
  },
  "guidance": {
    "summary": "string",
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

`GET /api/advisory/sse` emits one event per session change:

```
event: session
data: { ...AdvisorySessionRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `sessionId`. No polling.
