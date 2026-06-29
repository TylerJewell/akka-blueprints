# API contract — self-correcting-rag

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `{ "queryText": String, "requestedBy"?: String }` | `202 { "queryId": String }` | `RagEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (optional `?status=RETRIEVING\|GRADING\|...` — filtered client-side from `getAllQueries`) | `RagEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `RagEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` (one event per query change) | `RagEndpoint` ← `QueryView` |
| `POST` | `/api/corpus/documents` | `{ "docId": String, "title": String, "content": String, "source": String }` | `204` | `RagEndpoint` → `CorpusEntity` |
| `GET` | `/api/corpus/documents` | — | `200 [ Document... ]` | `RagEndpoint` ← `CorpusEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/queries`

- Missing `requestedBy` → `"anonymous"`.
- `queryText` must be non-empty and at most 2000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(queryText, requestedBy)`; the second submission returns the first `queryId` (200) instead of starting a new workflow.

## JSON shapes

### Query

```json
{
  "queryId": "q-8c1d4…",
  "queryText": "What drives soil carbon loss in drained peatlands?",
  "requestedBy": "researcher-1",
  "maxRewriteAttempts": 2,
  "maxGenerationAttempts": 2,
  "status": "ANSWERED",
  "retrievalPasses": [
    {
      "passNumber": 1,
      "retrieval": {
        "queryUsed": "What drives soil carbon loss in drained peatlands?",
        "documents": [
          { "docId": "doc-031", "title": "Peatland Carbon Dynamics", "content": "…", "source": "corpus" },
          { "docId": "doc-044", "title": "Water Table Effects on CO2 Flux", "content": "…", "source": "corpus" }
        ],
        "retrievedAt": "2026-06-28T09:01:00Z"
      },
      "grading": {
        "grades": [
          { "docId": "doc-031", "verdict": "RELEVANT", "score": 0.93, "rationale": "Directly addresses peatland carbon loss mechanisms." },
          { "docId": "doc-044", "verdict": "RELEVANT", "score": 0.88, "rationale": "Provides water table drawdown data linked to CO2 emissions." }
        ],
        "retainedDocuments": [ { "docId": "doc-031", "…": "…" }, { "docId": "doc-044", "…": "…" } ],
        "totalRetrieved": 2,
        "gradedAt": "2026-06-28T09:01:08Z"
      },
      "rewrite": null
    }
  ],
  "generatedAnswer": {
    "answerText": "Drainage of peatlands accelerates aerobic microbial decomposition… (doc-031). Water table drawdown is the primary driver… (doc-044).",
    "citedDocIds": ["doc-031", "doc-044"],
    "generatedAt": "2026-06-28T09:01:18Z"
  },
  "hallucinationVerdict": {
    "result": "GROUNDED",
    "rationale": "All claims are supported by doc-031 and doc-044.",
    "unsupportedClaims": [],
    "gradedAt": "2026-06-28T09:01:26Z"
  },
  "failureReason": null,
  "createdAt": "2026-06-28T09:00:57Z",
  "finishedAt": "2026-06-28T09:01:27Z"
}
```

### Document

```json
{
  "docId": "doc-031",
  "title": "Peatland Carbon Dynamics",
  "content": "Drainage of peatlands exposes previously anaerobic organic matter to aerobic conditions…",
  "source": "corpus"
}
```

### SSE event format

```
event: query-update
data: { "queryId": "q-8c1d4…", "status": "GRADING", "retrievalPasses": [...], ... }
```

One event per state transition. Clients reconcile by `queryId`. The full `Query` JSON is included so a fresh client can render the row without a separate fetch.

### Guardrail verdict codes

Hallucination grader returns one of two `result` values:

| Value | Meaning |
|---|---|
| `GROUNDED` | Every claim in the answer is supported by the retained documents. |
| `HALLUCINATED` | At least one claim in `unsupportedClaims` is not supported. |

### Relevance verdict codes

| Value | Meaning |
|---|---|
| `RELEVANT` | Document score is at or above `relevanceThreshold`. |
| `IRRELEVANT` | Document score is below `relevanceThreshold`. |
