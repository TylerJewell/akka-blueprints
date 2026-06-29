# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/research` | `{ "topic": "string" }` | `{ "briefId": "uuid" }` | `ResearchEndpoint` → `RequestQueue` |
| GET | `/api/research` | — | `{ "briefs": [ResearchBriefRow, ...] }` | `ResearchEndpoint` → `ResearchView` |
| GET | `/api/research?status=SYNTHESISED` | — | filtered list (client-side filter) | `ResearchEndpoint` |
| GET | `/api/research/{id}` | — | `ResearchBriefRow` or 404 | `ResearchEndpoint` |
| GET | `/api/research/sse` | — | `text/event-stream` of `ResearchBriefRow` | `ResearchEndpoint` → `ResearchView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ResearchEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ResearchEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ResearchEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/research` request:

```json
{ "topic": "How does the EU AI Act classify general-purpose models?" }
```

`ResearchBriefRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "briefId": "uuid",
  "topic": "string",
  "status": "PLANNING | IN_PROGRESS | SYNTHESISED | DEGRADED | BLOCKED",
  "findings": { "findings": [ { "headline": "...", "source": "...", "content": "..." } ], "gatheredAt": "ISO-8601" },
  "analysis": { "thesis": "...", "implications": ["..."], "analysedAt": "ISO-8601" },
  "synthesised": { "summary": "...", "guardrailVerdict": "ok", "synthesisedAt": "ISO-8601" },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/research/sse` emits one event per brief change:

```
event: brief
data: { ...ResearchBriefRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `briefId`. No polling.
