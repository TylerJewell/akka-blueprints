# API contract — code-search-rag-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first) | `QueryEndpoint` ← `ChunkIndexView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `ChunkIndexView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `ChunkIndexView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "questionText": "Where is the connection pool size configured?",
  "corpusTag": "akka-http",
  "retrievedChunks": [
    {
      "chunkId": "chunk-001",
      "filePath": "src/main/java/com/example/HttpServerApp.java",
      "startLine": 45,
      "endLine": 62,
      "language": "java",
      "content": "// HttpServerApp.java — server binding\npublic static void main(String[] args) {\n    ...\n    Http().newServerAt(host, port).bind(routes);\n}",
      "corpusTag": "akka-http"
    }
  ],
  "submittedBy": "developer-42"
}
```

### Query (response body)

```json
{
  "queryId": "q-7ac...",
  "request": {
    "queryId": "q-7ac...",
    "questionText": "Where is the connection pool size configured?",
    "corpusTag": "akka-http",
    "retrievedChunks": [
      {
        "chunkId": "chunk-001",
        "filePath": "src/main/java/com/example/HttpServerApp.java",
        "startLine": 45,
        "endLine": 62,
        "language": "java",
        "content": "(raw chunk content preserved for audit)",
        "corpusTag": "akka-http"
      }
    ],
    "submittedBy": "developer-42",
    "submittedAt": "2026-06-28T12:34:00Z"
  },
  "sanitized": {
    "redactedChunks": [
      {
        "chunkId": "chunk-001",
        "filePath": "src/main/java/com/example/HttpServerApp.java",
        "startLine": 45,
        "endLine": 62,
        "language": "java",
        "content": "// HttpServerApp.java — server binding\npublic static void main(String[] args) {\n    ...\n    Http().newServerAt(host, port).bind(routes);\n}",
        "corpusTag": "akka-http"
      }
    ],
    "secretCategoriesFound": []
  },
  "answer": {
    "answerText": "The HTTP server is bound in HttpServerApp.java at line 51 via Http().newServerAt(host, port).bind(routes). The port value is read from configuration at line 47.",
    "references": [
      {
        "filePath": "src/main/java/com/example/HttpServerApp.java",
        "startLine": 45,
        "endLine": 62,
        "language": "java",
        "relevanceBlurb": "Contains the server binding call and the config lookup for the port number."
      }
    ],
    "answeredAt": "2026-06-28T12:34:18Z"
  },
  "grounding": {
    "score": 5,
    "rationale": "All cited file paths appear in the retrieved chunk set; answer text is non-empty.",
    "evaluatedAt": "2026-06-28T12:34:19Z"
  },
  "status": "GROUNDED",
  "createdAt": "2026-06-28T12:34:00Z",
  "finishedAt": "2026-06-28T12:34:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

Raw chunk `content` is available on the full entity response at `request.retrievedChunks[*].content`. The view row omits chunk bodies to keep the list response compact; the UI fetches `/api/queries/{id}` when a developer needs to inspect raw content.

### SSE event format

```
event: query-update
data: { "queryId": "q-7ac...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `CHUNKS_SANITIZED`, `ANSWERING`, `ANSWERED`, `GROUNDED`, `FAILED`). Each event carries the full row at the moment of transition so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
