# API contract — graphrag-assistant

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/ask` | `{ "question": "string" }` | `{ "queryId": "uuid" }` | QueryEndpoint → QueryEntity, ResearchAgent |
| GET | `/api/queries` | — | `{ "queries": [Query, ...] }` | QueryEndpoint → QueriesView |
| GET | `/api/queries/{id}` | — | `Query` | QueryEndpoint → QueriesView |
| GET | `/api/queries/sse` | — | SSE stream of `Query` | QueryEndpoint → QueriesView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | QueryEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | QueryEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | QueryEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Query` (lifecycle fields nullable → `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "question": "string",
  "status": "RECEIVED | ANSWERED | BLOCKED",
  "createdAt": "ISO-8601",
  "scope": "local | global | null",
  "chunkCount": "int or null",
  "retrievedAt": "ISO-8601 or null",
  "answer": "string or null",
  "grounded": "bool or null",
  "citations": ["doc-id", "..."],
  "answeredAt": "ISO-8601 or null",
  "blockedReason": "string or null"
}
```

`Answer` (ResearchAgent return type, internal):

```json
{ "text": "string", "scope": "local | global", "grounded": true, "citations": ["doc-id"], "chunkCount": 2 }
```

`RetrievalResult` (CorpusIndex search return type, internal):

```json
{ "chunks": ["string", "..."], "citations": ["doc-id", "..."] }
```

## SSE event format

`GET /api/queries/sse` emits one `data:` line per updated `Query`, JSON-encoded as
above, on each entity transition (created, answered, blocked). Produced by
`serverSentEventsForView` over `QueriesView.getAllQueries`.
