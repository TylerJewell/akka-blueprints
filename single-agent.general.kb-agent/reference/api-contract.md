# API contract — kb-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `POST` | `/api/documents` | `IndexDocumentRequest` | `201 { documentId }` | `QueryEndpoint` → `KbDocumentConsumer` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first) | `QueryEndpoint` ← `KbView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `KbView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `KbView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "questionText": "What is the data retention period for customer records?",
  "submittedBy": "user-42"
}
```

### IndexDocumentRequest (request body)

```json
{
  "documentId": "doc-policies-007",
  "title": "Data Retention Policy v2.3",
  "body": "Customer records shall be retained for a period of 7 years from the date of last transaction. Verified deletion requests will be fulfilled within thirty (30) calendar days...",
  "collection": "policies"
}
```

### Query (response body)

```json
{
  "queryId": "q-4ac...",
  "questionText": "What is the data retention period for customer records?",
  "submittedBy": "user-42",
  "passages": [
    {
      "passageId": "p-001",
      "documentId": "doc-policies-007",
      "documentTitle": "Data Retention Policy v2.3",
      "excerpt": "Customer records shall be retained for a period of 7 years from the date of last transaction.",
      "relevanceScore": 0.87
    },
    {
      "passageId": "p-002",
      "documentId": "doc-policies-007",
      "documentTitle": "Data Retention Policy v2.3",
      "excerpt": "Verified deletion requests will be fulfilled within thirty (30) calendar days.",
      "relevanceScore": 0.74
    }
  ],
  "answer": {
    "answer": "The data retention period for customer records is 7 years from the date of last transaction. Deletion requests are processed within 30 days.",
    "citations": [
      {
        "passageId": "p-001",
        "documentTitle": "Data Retention Policy v2.3",
        "excerpt": "Customer records shall be retained for a period of 7 years from the date of last transaction."
      },
      {
        "passageId": "p-002",
        "documentTitle": "Data Retention Policy v2.3",
        "excerpt": "Verified deletion requests will be fulfilled within thirty (30) calendar days."
      }
    ],
    "answerStatus": "GROUNDED",
    "answeredAt": "2026-06-28T14:00:18Z"
  },
  "groundedness": {
    "score": 5,
    "rationale": "All answer sentences trace to retrieved passages; both citations are valid.",
    "citedSentences": 2,
    "totalSentences": 2,
    "evaluatedAt": "2026-06-28T14:00:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: query-update
data: { "queryId": "q-4ac...", "status": "ANSWER_RECORDED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RETRIEVING`, `ANSWERING`, `ANSWER_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
