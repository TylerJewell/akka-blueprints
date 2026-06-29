# API contract — multi-step-query-engine

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ QueryRecord... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 QueryRecord` / `404` | `QueryEndpoint` ← `QueryView` |
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
  "question": "What are the primary drivers of transformer model scaling laws?"
}
```

### QueryRecord (response body)

```json
{
  "queryId": "q-3bc7e1...",
  "question": "What are the primary drivers of transformer model scaling laws?",
  "decomposition": {
    "subQuestions": [
      { "subQuestionId": "sq-compute", "text": "What computational factors determine transformer scaling behaviour?", "priority": 1 },
      { "subQuestionId": "sq-data", "text": "What role does training-data volume play in scaling law predictions?", "priority": 2 }
    ],
    "decomposedAt": "2026-06-28T10:00:00Z"
  },
  "evidence": {
    "passages": [
      {
        "passageId": "p-0a1b2c3d",
        "subQuestionId": "sq-compute",
        "source": "Chinchilla scaling study",
        "url": "https://example.org/chinchilla-2022",
        "text": "Model loss scales as a power law of compute budget when parameters and tokens are co-optimised.",
        "relevanceScore": 0.93
      },
      {
        "passageId": "p-4e5f6a7b",
        "subQuestionId": "sq-data",
        "source": "Scaling laws for neural language models",
        "url": "https://example.org/kaplan-2020",
        "text": "Dataset size exhibits diminishing returns relative to parameter count; underfitting on data constrains maximum achievable loss.",
        "relevanceScore": 0.88
      }
    ],
    "retrievedAt": "2026-06-28T10:00:08Z"
  },
  "answer": {
    "question": "What are the primary drivers of transformer model scaling laws?",
    "summary": "Compute budget and training-data volume are the two principal drivers.",
    "confidence": "HIGH",
    "sections": [
      {
        "subQuestionId": "sq-compute",
        "heading": "Computational factors in scaling",
        "body": "The Chinchilla study demonstrates that transformer loss follows a power law of total compute when parameter count and training tokens are jointly optimised.",
        "citedPassageIds": ["p-0a1b2c3d"]
      },
      {
        "subQuestionId": "sq-data",
        "heading": "Training-data volume in scaling predictions",
        "body": "Kaplan et al. show that dataset size has diminishing returns relative to parameter count.",
        "citedPassageIds": ["p-4e5f6a7b"]
      }
    ],
    "synthesizedAt": "2026-06-28T10:00:15Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Sub-question coverage, citation provenance, confidence calibration, and section parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:15Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:15Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### QueryRecord — hallucinated-citation example (eval score ≤ 2)

```json
{
  "queryId": "q-7fa2d8...",
  "status": "EVALUATED",
  "eval": {
    "score": 2,
    "rationale": "Citation provenance failed: section 'sq-compute' cites passageId 'p-xxxxxxxx' which does not appear in the retrieved EvidenceSet.",
    "evaluatedAt": "2026-06-28T10:05:00Z"
  }
}
```

### SSE event format

```
event: query-update
data: { "queryId": "q-3bc7e1...", "status": "RETRIEVED", "evidence": { ... }, ... }
```

One event per state transition (`CREATED`, `DECOMPOSING`, `DECOMPOSED`, `RETRIEVING`, `RETRIEVED`, `SYNTHESIZING`, `SYNTHESIZED`, `EVALUATED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitQueryRequest` record and the `QueryCreated` event to carry it.
