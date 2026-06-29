# API contract — rag-citation-answerer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/documents` | `UploadDocumentRequest` | `201 { documentId }` | `QueryEndpoint` → `DocumentEntity` |
| `GET` | `/api/documents` | — | `200 [ DocumentRow... ]` (newest-first) | `QueryEndpoint` ← `DocumentView` |
| `GET` | `/api/documents/{id}` | — | `200 DocumentRow` / `404` | `QueryEndpoint` ← `DocumentView` |
| `GET` | `/api/documents/sse` | — | `text/event-stream` | `QueryEndpoint` ← `DocumentView` |
| `POST` | `/api/queries` | `AskQuestionRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity`, `AnswerWorkflow` |
| `GET` | `/api/queries` | — | `200 [ QueryRow... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 QueryRow` / `404` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `QueryEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `QueryEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `QueryEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### UploadDocumentRequest (request body for `POST /api/documents`)

```json
{
  "documentTitle": "Synthetic Climate Policy Brief 2024",
  "rawText": "This policy brief outlines the framework agreed by signatory nations ...",
  "uploadedBy": "analyst-42"
}
```

### DocumentRow (response body for `GET /api/documents` and `GET /api/documents/{id}`)

```json
{
  "documentId": "d-3a7f...",
  "documentTitle": "Synthetic Climate Policy Brief 2024",
  "sanitized": {
    "redactedText": "This policy brief outlines the framework agreed by [REDACTED-ORG] and [REDACTED-NAME] ...",
    "piiCategoriesFound": ["person-name", "email"]
  },
  "status": "INDEXED",
  "chunkCount": 14,
  "createdAt": "2026-06-28T10:00:00Z",
  "indexedAt": "2026-06-28T10:00:03Z"
}
```

Note: `upload.rawText` is omitted from the DocumentRow returned by the view. Audit access to the raw text requires a separate admin endpoint that deployers add during customisation.

### AskQuestionRequest (request body for `POST /api/queries`)

```json
{
  "questionText": "What emission reduction targets does the 2024 climate policy brief describe?",
  "documentIds": ["d-3a7f..."],
  "askedBy": "analyst-42"
}
```

### QueryRow (response body for `GET /api/queries` and `GET /api/queries/{id}`)

```json
{
  "queryId": "q-8bc2...",
  "questionText": "What emission reduction targets does the 2024 climate policy brief describe?",
  "documentIds": ["d-3a7f..."],
  "askedBy": "analyst-42",
  "retrieval": {
    "chunks": [
      {
        "chunkId": "doc-d-3a7f-3",
        "documentId": "d-3a7f...",
        "documentTitle": "Synthetic Climate Policy Brief 2024",
        "chunkIndex": 3,
        "text": "Parties commit to a 45 percent reduction in greenhouse gas emissions by 2030 ...",
        "sectionHint": "§2"
      }
    ],
    "totalChunksSearched": 14
  },
  "answer": {
    "answerText": "The policy brief sets a 45% reduction in greenhouse gas emissions by 2030 relative to 2005 levels, with a net-zero target by 2050.",
    "confidence": "HIGH",
    "citations": [
      {
        "chunkId": "doc-d-3a7f-3",
        "documentId": "d-3a7f...",
        "documentTitle": "Synthetic Climate Policy Brief 2024",
        "excerpt": "Parties commit to a 45 percent reduction in greenhouse gas emissions by 2030, measured against a 2005 baseline.",
        "sectionHint": "§2"
      }
    ],
    "answeredAt": "2026-06-28T10:05:22Z"
  },
  "status": "ANSWERED",
  "createdAt": "2026-06-28T10:05:10Z",
  "answeredAt": "2026-06-28T10:05:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

Two SSE streams run in parallel:

**Document stream** (`GET /api/documents/sse`):
```
event: document-update
data: { "documentId": "d-3a7f...", "status": "INDEXED", "chunkCount": 14, ... }
```

**Query stream** (`GET /api/queries/sse`):
```
event: query-update
data: { "queryId": "q-8bc2...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per state transition. Clients reconcile by `documentId` or `queryId`; each event carries the full row at the moment of transition so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap `QueryEndpoint` with their auth middleware and set `uploadedBy` / `askedBy` from the authenticated principal rather than the request body. Document access should be scoped per tenant once multi-tenancy is introduced.
