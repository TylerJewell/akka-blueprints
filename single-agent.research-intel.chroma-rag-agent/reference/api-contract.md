# API contract — chroma-rag-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first) | `QueryEndpoint` ← `RagView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `RagView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `RagView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "questionText": "How does an Akka EventSourcedEntity rebuild its state after a restart?",
  "submittedBy": "developer-42"
}
```

### Query (response body)

```json
{
  "queryId": "q-7ac...",
  "request": {
    "queryId": "q-7ac...",
    "questionText": "How does an Akka EventSourcedEntity rebuild its state after a restart?",
    "submittedBy": "developer-42",
    "submittedAt": "2026-06-28T14:22:00Z"
  },
  "retrievedChunks": [
    {
      "chunkId": "akka-doc-007",
      "documentTitle": "Akka Event Sourced Entities",
      "passage": "An EventSourcedEntity processes incoming commands and responds by emitting one or more events. These events are durably stored and replayed to restore state after a restart.",
      "score": 0.94
    },
    {
      "chunkId": "akka-doc-012",
      "documentTitle": "Akka Event Sourced Entities",
      "passage": "The emptyState() method returns the initial state of the entity before any events have been applied.",
      "score": 0.87
    }
  ],
  "answer": {
    "responseText": "An EventSourcedEntity rebuilds its state by replaying all stored events from the beginning, applying each event-applier method in sequence starting from the initial state returned by emptyState().",
    "citations": [
      {
        "chunkId": "akka-doc-007",
        "documentTitle": "Akka Event Sourced Entities",
        "passage": "An EventSourcedEntity processes incoming commands and responds by emitting one or more events. These events are durably stored and replayed to restore state after a restart."
      },
      {
        "chunkId": "akka-doc-012",
        "documentTitle": "Akka Event Sourced Entities",
        "passage": "The emptyState() method returns the initial state of the entity before any events have been applied."
      }
    ],
    "chunksRetrieved": 5,
    "answeredAt": "2026-06-28T14:22:18Z"
  },
  "groundedness": {
    "score": 5,
    "rationale": "All citations have matching chunkIds in the retrieved context and non-empty passages; responseText is directly traceable.",
    "evaluatedAt": "2026-06-28T14:22:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-7ac...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RETRIEVING`, `ANSWERING`, `ANSWERED`, `EVALUATED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
