# API contract — pgvector-rag-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/questions` | `AskQuestionRequest` | `201 { questionId }` | `QueryEndpoint` → `QuestionEntity` |
| `GET` | `/api/questions` | — | `200 [ Question... ]` (newest-first) | `QueryEndpoint` ← `QuestionView` |
| `GET` | `/api/questions/{id}` | — | `200 Question` / `404` | `QueryEndpoint` ← `QuestionView` |
| `GET` | `/api/questions/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QuestionView` |
| `POST` | `/api/corpus` | `IngestDocumentRequest` | `201 { documentId }` | `QueryEndpoint` → `CorpusEntity` |
| `GET` | `/api/corpus` | — | `200 [ CorpusDocument... ]` | `QueryEndpoint` ← `CorpusEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### AskQuestionRequest (request body)

```json
{
  "questionText": "How does emptyState() work in an EventSourcedEntity?",
  "askedBy": "developer-42"
}
```

### IngestDocumentRequest (request body)

```json
{
  "sourceLabel": "akka-event-sourcing-guide",
  "body": "An EventSourcedEntity persists state by appending events..."
}
```

### Question (response body)

```json
{
  "questionId": "q-7a3f...",
  "questionText": "How does emptyState() work in an EventSourcedEntity?",
  "askedBy": "developer-42",
  "answer": {
    "answerText": "The emptyState() method returns the entity's initial state before any events have been applied [evt-src-002]. It must not reference commandContext(), as no command context exists at that point [evt-src-007].",
    "citations": [
      { "chunkId": "evt-src-002", "sourceLabel": "akka-event-sourcing-guide" },
      { "chunkId": "evt-src-007", "sourceLabel": "akka-event-sourcing-guide" }
    ],
    "passages": [
      {
        "chunkId": "evt-src-002",
        "sourceLabel": "akka-event-sourcing-guide",
        "text": "The emptyState() method returns the initial state...",
        "relevanceScore": 0.91
      }
    ],
    "answeredAt": "2026-06-28T15:00:18Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Every factual claim cites a chunk; answer length is proportionate to retrieved passages.",
    "evaluatedAt": "2026-06-28T15:00:19Z"
  },
  "status": "EVALUATED",
  "submittedAt": "2026-06-28T15:00:00Z",
  "finishedAt": "2026-06-28T15:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### CorpusDocument (response body)

```json
{
  "documentId": "doc-a1b2...",
  "sourceLabel": "akka-event-sourcing-guide",
  "status": "INDEXED",
  "chunkCount": 12,
  "addedAt": "2026-06-28T14:50:00Z"
}
```

### SSE event format

```
event: question-update
data: { "questionId": "q-7a3f...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RETRIEVING`, `ANSWERING`, `ANSWERED`, `EVALUATED`, `FAILED`). Clients reconcile by `questionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `askedBy` from the authenticated principal rather than the request body.
