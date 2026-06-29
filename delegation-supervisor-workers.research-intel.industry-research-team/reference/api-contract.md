# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/research-request` | `{ "request": "string" }` | `{ "briefId": "uuid" }` | ResearchEndpoint → InboundRequestQueue |
| GET | `/api/briefs?status=...` | — | `{ "briefs": [Brief, ...] }` | ResearchEndpoint → BriefsView |
| GET | `/api/briefs/{briefId}` | — | `Brief` or 404 | ResearchEndpoint → BriefsView |
| GET | `/api/briefs/sse` | — | SSE stream of `Brief` | ResearchEndpoint → BriefsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ResearchEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ResearchEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ResearchEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

The `status` query parameter is filtered client-side in the endpoint from `BriefsView.getAllBriefs` — the view has no `WHERE status` query because Akka cannot auto-index enum columns (Lesson 2).

## Payload shapes

`Brief` (lifecycle fields are nullable — `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "request": "string or null",
  "status": "PLANNED | ANALYZING | SYNTHESIZED | BLOCKED | SANITIZED | COMPLETED",
  "sectors": ["string", "..."],
  "findings": [
    { "sector": "string", "summary": "string", "signals": ["string", "..."] }
  ],
  "plannedAt": "ISO-8601 or null",
  "briefTitle": "string or null",
  "analyzedAt": "ISO-8601 or null",
  "briefBody": "string or null",
  "synthesizedAt": "ISO-8601 or null",
  "groundingScore": 0.0,
  "blockedReason": "string or null",
  "blockedAt": "ISO-8601 or null",
  "sanitizedBody": "string or null",
  "sanitizedAt": "ISO-8601 or null",
  "completedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/briefs/sse` emits `text/event-stream`. Each event's `data:` line is a single serialized `Brief` JSON object (same form as above), one event per view-row update. The UI replaces the matching row by `id`.
