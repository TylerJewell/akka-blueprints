# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/research` | `{ "question": "string" }` | `{ "runId": "uuid" }` | `ResearchEndpoint` → `QuestionQueue` |
| GET | `/api/research` | — | `{ "runs": [ResearchRunRow, ...] }` | `ResearchEndpoint` → `ResearchView` |
| GET | `/api/research?status=ANSWERED` | — | filtered list (client-side filter) | `ResearchEndpoint` |
| GET | `/api/research/{id}` | — | `ResearchRunRow` or 404 | `ResearchEndpoint` |
| GET | `/api/research/sse` | — | `text/event-stream` of `ResearchRunRow` | `ResearchEndpoint` → `ResearchView` |
| GET | `/api/research/monitoring` | — | `{ "flaggedCount": int, "lastFlaggedAt": "ISO-8601 or null" }` | `ResearchEndpoint` → `ResearchView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ResearchEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ResearchEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ResearchEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/research` request:

```json
{ "question": "What governance obligations apply to general-purpose AI models under the EU AI Act?" }
```

`ResearchRunRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "runId": "uuid",
  "question": "string",
  "status": "QUEUED | IN_PROGRESS | ANSWERED | DEGRADED | BLOCKED",
  "webContent": {
    "pages": [
      { "url": "https://...", "title": "...", "extractedText": "...", "fetchedAt": "ISO-8601" }
    ],
    "piiSanitized": true,
    "gatheredAt": "ISO-8601"
  },
  "docContent": {
    "documentRef": "string",
    "sections": [
      { "heading": "...", "content": "..." }
    ],
    "piiSanitized": true,
    "readAt": "ISO-8601"
  },
  "answer": {
    "summary": "string",
    "citations": [
      { "text": "...", "sourceUrl": "https://...", "sourceDocument": "string or null" }
    ],
    "piiSanitized": true,
    "synthesisedAt": "ISO-8601"
  },
  "blockedUrl": "string or null",
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "oversightFlagged": false,
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

`GET /api/research/monitoring` response:

```json
{
  "flaggedCount": 3,
  "lastFlaggedAt": "2026-06-28T14:22:00Z"
}
```

## SSE event format

`GET /api/research/sse` emits one event per run change:

```
event: run
data: { ...ResearchRunRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `runId`. No polling.
