# API contract — corrective-rag-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `{ "question": String, "relevanceThresholdOverride"?: Double, "requestedBy"?: String }` | `202 { "queryId": String }` | `RagEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (optional `?status=RETRIEVING\|EVALUATING\|WEB_SEARCHING\|ANSWERED\|ANSWER_DEGRADED` — filtered client-side from `getAllQueries`) | `RagEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `RagEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` (one event per query change) | `RagEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/queries`

- Missing `relevanceThresholdOverride` → 0.7 (from `corrective-rag.retrieval.default-threshold`).
- Missing `requestedBy` → `"anonymous"`.
- `relevanceThresholdOverride` must be in `[0.1, 1.0]`; otherwise `400`.
- `question` must be non-empty and under 2000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(question, requestedBy)`; the second submission returns the first `queryId` (200) instead of starting a new workflow.

## JSON shapes

### Query

```json
{
  "queryId": "q-3c9a1…",
  "question": "What are the NIST AI RMF governance requirements?",
  "relevanceThreshold": 0.7,
  "maxRetrievalAttempts": 3,
  "status": "ANSWERED",
  "attempts": [
    {
      "attemptNumber": 1,
      "retrieval": {
        "chunks": [
          {
            "chunkId": "nist-ai-rmf-govern-001",
            "excerpt": "The GOVERN function of the NIST AI RMF establishes organizational policies …",
            "score": 0.84
          }
        ],
        "retrievedAt": "2026-06-28T09:00:05Z"
      },
      "verdict": {
        "judgment": "SUFFICIENT",
        "scoredChunks": [
          {
            "chunkId": "nist-ai-rmf-govern-001",
            "excerpt": "The GOVERN function of the NIST AI RMF establishes organizational policies …",
            "score": 0.84
          }
        ],
        "averageScore": 0.84,
        "rationale": "Retrieved chunk directly addresses NIST AI RMF governance requirements.",
        "evaluatedAt": "2026-06-28T09:00:11Z"
      },
      "guardrail": null,
      "webSearch": null
    }
  ],
  "answer": {
    "text": "The NIST AI RMF's GOVERN function requires organizations to establish AI risk governance policies, assign accountability roles, and integrate risk management into enterprise processes [nist-ai-rmf-govern-001].",
    "citedChunkIds": ["nist-ai-rmf-govern-001"],
    "citedUrls": [],
    "source": "RETRIEVAL_ONLY",
    "generatedAt": "2026-06-28T09:00:18Z"
  },
  "degradationReason": null,
  "createdAt": "2026-06-28T09:00:02Z",
  "finishedAt": "2026-06-28T09:00:19Z"
}
```

### Guardrail verdict form

```json
{ "cleared": true,  "reasonCode": "OK",                  "detail": "" }
{ "cleared": false, "reasonCode": "DISALLOWED_PATTERN",  "detail": "Query matches disallowed pattern: \\bSSN\\b" }
```

`reasonCode` is one of `OK`, `DISALLOWED_PATTERN`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: query-update
data: { "queryId": "q-3c9a1…", "status": "EVALUATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `queryId`. The full `Query` JSON is included so a fresh client can render the row without a separate fetch.
