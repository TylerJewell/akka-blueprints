# API contract — bm25-rag-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "questionText": "How does scaled dot-product attention work?",
  "topK": 5,
  "submittedBy": "developer-42"
}
```

### Query (response body)

```json
{
  "queryId": "q-7a3...",
  "request": {
    "queryId": "q-7a3...",
    "questionText": "How does scaled dot-product attention work?",
    "topK": 5,
    "submittedBy": "developer-42",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "passages": {
    "passages": [
      {
        "passageId": "p-014",
        "title": "Scaled Dot-Product Attention",
        "snippet": "We scale the dot products by 1/sqrt(d_k) to counteract the effect of large magnitudes...",
        "bm25Score": 12.4
      },
      {
        "passageId": "p-031",
        "title": "Attention Mechanism Formula",
        "snippet": "Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) V",
        "bm25Score": 9.8
      }
    ],
    "totalDocsScanned": 80
  },
  "answer": {
    "answerType": "DIRECT",
    "answerText": "Scaled dot-product attention computes compatibility between a query vector and a set of key vectors by taking their dot product, scaling by the square root of the key dimension, and applying softmax to weight the value vectors.",
    "citations": [
      {
        "passageId": "p-014",
        "quotedFragment": "We scale the dot products by 1/sqrt(d_k) to counteract the effect of large magnitudes when the dimensionality d_k is large, which pushes the softmax into regions with extremely small gradients."
      },
      {
        "passageId": "p-031",
        "quotedFragment": "Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) V"
      }
    ],
    "decidedAt": "2026-06-28T14:00:18Z"
  },
  "grounding": {
    "score": 5,
    "rationale": "All citations contain verbatim excerpts from delivered passages; DIRECT answer with at least one citation.",
    "scoredAt": "2026-06-28T14:00:19Z"
  },
  "status": "SCORED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-7a3...", "status": "ANSWER_RECORDED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `PASSAGES_ATTACHED`, `ANSWERING`, `ANSWER_RECORDED`, `SCORED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
