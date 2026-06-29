# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/digests` | `{ "query": "string" }` | `{ "digestId": "uuid" }` | `DigestEndpoint` → `QueryQueue` |
| GET | `/api/digests` | — | `{ "digests": [ResearchDigestRow, ...] }` | `DigestEndpoint` → `DigestView` |
| GET | `/api/digests?status=SYNTHESISED` | — | filtered list (client-side filter) | `DigestEndpoint` |
| GET | `/api/digests/{id}` | — | `ResearchDigestRow` or 404 | `DigestEndpoint` |
| GET | `/api/digests/sse` | — | `text/event-stream` of `ResearchDigestRow` | `DigestEndpoint` → `DigestView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `DigestEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `DigestEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `DigestEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/digests` request:

```json
{ "query": "What are the latest breakthroughs in quantum error correction?" }
```

`ResearchDigestRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "digestId": "uuid",
  "query": "string",
  "status": "QUEUED | SCANNING | SYNTHESISED | DEGRADED | BLOCKED",
  "publications": {
    "publications": [
      { "title": "...", "doi": "10.xxxx/...", "venue": "...", "abstract_": "..." }
    ],
    "scoutedAt": "ISO-8601"
  },
  "trends": {
    "thesis": "...",
    "emergingAreas": ["..."],
    "gaps": ["..."],
    "analysedAt": "ISO-8601"
  },
  "synthesised": {
    "summary": "...",
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

`GET /api/digests/sse` emits one event per digest change:

```
event: digest
data: { ...ResearchDigestRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `digestId`. No polling.
