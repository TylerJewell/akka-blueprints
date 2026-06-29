# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/query` | `{ "question": "string" }` | `{ "sessionId": "uuid" }` | `QueryEndpoint` → `IndexCallQueue` |
| GET | `/api/query` | — | `{ "sessions": [QuerySessionRow, ...] }` | `QueryEndpoint` → `QuerySessionView` |
| GET | `/api/query?status=SYNTHESISED` | — | filtered list (client-side filter) | `QueryEndpoint` |
| GET | `/api/query/{id}` | — | `QuerySessionRow` or 404 | `QueryEndpoint` |
| GET | `/api/query/sse` | — | `text/event-stream` of `QuerySessionRow` | `QueryEndpoint` → `QuerySessionView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `QueryEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `QueryEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `QueryEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/query` request:

```json
{ "question": "What changed in the authentication API between v2 and v3, and does the FAQ cover migration?" }
```

`QuerySessionRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "sessionId": "uuid",
  "question": "string",
  "status": "DECOMPOSING | RETRIEVING | SYNTHESISED | PARTIAL | BLOCKED",
  "decomposition": {
    "subQuestions": [
      { "text": "string", "indexId": "docs | faq | changelog | api-ref" }
    ],
    "decomposedAt": "ISO-8601"
  },
  "indexResults": [
    {
      "subQuestion": "string",
      "indexId": "string",
      "answer": "string",
      "sourceRefs": [{ "title": "string", "uri": "string" }],
      "retrievedAt": "ISO-8601"
    }
  ],
  "combinedAnswer": {
    "summary": "string",
    "indexResults": [ "... same shape as above ..." ],
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

`GET /api/query/sse` emits one event per session change:

```
event: session
data: { ...QuerySessionRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `sessionId`. No polling.
