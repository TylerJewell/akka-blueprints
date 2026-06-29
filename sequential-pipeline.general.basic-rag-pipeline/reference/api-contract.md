# API contract — basic-rag-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `SubmitQueryRequest` | `201 { sessionId }` | `RagEndpoint` → `RagSessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ RagSessionRecord... ]` (newest-first) | `RagEndpoint` ← `RagSessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 RagSessionRecord` / `404` | `RagEndpoint` ← `RagSessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `RagEndpoint` ← `RagSessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `RagEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `RagEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `RagEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "question": "What are the key principles of transformer architectures?"
}
```

### RagSessionRecord (response body)

```json
{
  "sessionId": "s-3bc7a1...",
  "question": "What are the key principles of transformer architectures?",
  "corpus": {
    "corpusId": "default",
    "chunks": [
      {
        "chunkId": "ck-a1b2c3d4",
        "docId": "doc-transformer-001",
        "text": "Self-attention allows each token in a sequence to attend to every other token.",
        "sourceUrl": "https://example.org/docs/transformer-architectures",
        "chunkIndex": 0
      }
    ],
    "indexedAt": "2026-06-28T10:00:00Z"
  },
  "answer": {
    "question": "What are the key principles of transformer architectures?",
    "answerText": "Transformer architectures rely on self-attention to capture long-range dependencies and positional encodings to inject sequence order without recurrence.",
    "citations": [
      {
        "chunkId": "ck-a1b2c3d4",
        "sourceUrl": "https://example.org/docs/transformer-architectures",
        "excerpt": "Self-attention allows each token in a sequence to attend to every other token."
      }
    ],
    "answeredAt": "2026-06-28T10:00:10Z"
  },
  "guardrailOutcome": {
    "verdict": "PASSED",
    "reason": "All citations grounded in indexed corpus.",
    "evaluatedAt": "2026-06-28T10:00:10Z"
  },
  "status": "ANSWERED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:10Z"
}
```

A session in `BLOCKED` state carries `guardrailOutcome.verdict = "BLOCKED"` and a non-null `reason` naming the first ungrounded citation URL. The `answer` field holds the last draft produced (for debugging); the `status` makes clear it was not accepted.

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-3bc7a1...", "status": "INDEXED", "corpus": { ... }, ... }
```

One event per state transition (`CREATED`, `INDEXING`, `INDEXED`, `ANSWERING`, `ANSWERED`, `BLOCKED`, `FAILED`) and one per guardrail evaluation:

```
event: session-guardrail
data: { "sessionId": "s-3bc7a1...", "verdict": "BLOCKED", "reason": "citation-not-grounded: https://example.org/not-indexed not in corpus", "evaluatedAt": "..." }
```

Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitQueryRequest` record and the `SessionCreated` event to carry it.
