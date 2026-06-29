# API contract — rag-knowledge-agent

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Lifecycle fields that are null before their transition serialize as raw value or `null` (Jackson `Optional<T>` handling).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/ask` | `{ "question": "string" }` | `{ "sessionId": "uuid" }` | KnowledgeEndpoint |
| GET | `/api/sessions?status=...` | — | `{ "sessions": [QuerySession, ...] }` | KnowledgeEndpoint |
| GET | `/api/sessions/{id}` | — | `QuerySession` \| 404 | KnowledgeEndpoint |
| GET | `/api/sessions/sse` | — | SSE stream of `QuerySession` | KnowledgeEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | KnowledgeEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | KnowledgeEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | KnowledgeEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

The `status` query parameter is filtered client-side in the endpoint after `QuerySessionView.getAllSessions` (Akka cannot auto-index the enum column).

## Payload shapes

`QuerySession`:

```json
{
  "id": "uuid",
  "question": "string",
  "status": "RECEIVED | RETRIEVED | ANSWERED | REFUSED | EVALUATED",
  "receivedAt": "ISO-8601",
  "retrievedChunks": [Citation],
  "retrievedAt": "ISO-8601 or null",
  "answer": "string or null",
  "citations": [Citation],
  "grounded": "boolean or null",
  "answeredAt": "ISO-8601 or null",
  "refusalReason": "string or null",
  "faithfulnessScore": "number 0..1 or null",
  "faithfulnessVerdict": "supported | partial | unsupported or null",
  "evaluatedAt": "ISO-8601 or null"
}
```

`Citation`:

```json
{ "docId": "string", "docTitle": "string", "chunkId": "string", "snippet": "string", "score": 0.0 }
```

## SSE event format

`GET /api/sessions/sse` emits one event per `QuerySession` change, projected from `QuerySessionView`:

```
event: session
data: { ...QuerySession... }
```

The App UI updates the matching row in place keyed by `id`.
