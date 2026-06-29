# API contract — rag-pdf-chat

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/documents` | `UploadDocumentRequest` | `201 { documentId }` | `ChatEndpoint` → `PdfDocumentEntity` |
| `GET` | `/api/documents` | — | `200 [ PdfDocument... ]` (newest-first) | `ChatEndpoint` ← `ChatSessionView` |
| `GET` | `/api/documents/{id}` | — | `200 PdfDocument` / `404` | `ChatEndpoint` ← `PdfDocumentEntity` |
| `POST` | `/api/documents/{id}/chat` | `AskQuestionRequest` | `201 { questionId }` | `ChatEndpoint` → `ChatSessionWorkflow` |
| `GET` | `/api/documents/{id}/chat` | — | `200 [ ChatExchange... ]` (newest-first) | `ChatEndpoint` ← `ChatSessionView` |
| `GET` | `/api/documents/{id}/chat/sse` | — | `text/event-stream` | `ChatEndpoint` ← `ChatSessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### UploadDocumentRequest (request body)

```json
{
  "filename": "distributed-systems-whitepaper.pdf",
  "pdfBase64": "JVBERi0xLjQK...",
  "sessionId": "session-abc123"
}
```

### AskQuestionRequest (request body)

```json
{
  "questionText": "How does Raft handle leader election when the current leader fails?",
  "sessionId": "session-abc123"
}
```

### PdfDocument (response body — list entry)

```json
{
  "documentId": "doc-7f4e...",
  "metadata": {
    "documentId": "doc-7f4e...",
    "filename": "distributed-systems-whitepaper.pdf",
    "totalPages": 8,
    "passageCount": 24,
    "uploadedAt": "2026-06-28T14:10:00Z"
  },
  "passages": null,
  "status": "INDEXED",
  "createdAt": "2026-06-28T14:10:00Z"
}
```

`passages` is `null` on list responses (omitted for size). The full passage list is returned on `GET /api/documents/{id}`.

### ChatExchange (response body)

```json
{
  "questionId": "q-3a8c...",
  "question": {
    "questionId": "q-3a8c...",
    "documentId": "doc-7f4e...",
    "sessionId": "session-abc123",
    "questionText": "How does Raft handle leader election when the current leader fails?",
    "askedAt": "2026-06-28T14:22:00Z"
  },
  "retrievedPassages": [
    {
      "passageId": "P-005",
      "pageNumber": 2,
      "chunkIndex": 4,
      "text": "Leader election in Raft requires a majority vote. A candidate that receives votes from a majority of the cluster becomes the new leader.",
      "termVector": null
    }
  ],
  "answer": {
    "answerable": true,
    "answerText": "When the current leader fails, Raft triggers a new election: any follower that times out without a heartbeat becomes a candidate and requests votes from the cluster.",
    "citations": [
      {
        "passageId": "P-005",
        "excerpt": "A candidate that receives votes from a majority of the cluster becomes the new leader.",
        "pageNumber": 2
      }
    ],
    "answeredAt": "2026-06-28T14:22:18Z"
  },
  "status": "ANSWERED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:18Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `termVector` on `PdfPassage` is always `null` in API responses — it is an in-process field only.

### SSE event format

```
event: exchange-update
data: { "questionId": "q-3a8c...", "documentId": "doc-7f4e...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per state transition (`RETRIEVING`, `ANSWERING`, `ANSWERED`, `UNANSWERABLE`, `FAILED`). Clients reconcile by `questionId` + `documentId`; an event always carries the full exchange row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `sessionId` from the authenticated principal rather than the request body.
