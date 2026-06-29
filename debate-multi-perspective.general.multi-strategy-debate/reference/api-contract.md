# API contract — multi-strategy-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `{ "question": String, "submittedBy"?: String }` | `202 { "queryId": String }` | `QueryEndpoint` → `QueryQueue` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries?status=SYNTHESIZED` | — | `200 [ Query... ]` (filtered client-side) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` (one event per query change) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

Status filtering (`?status=`) is applied client-side — the view returns all rows and the endpoint filters the list before serializing.

## JSON shapes

### Query

```json
{
  "queryId": "qy-3a1…",
  "question": "What is the difference between mutable and immutable data structures?",
  "status": "SYNTHESIZED",
  "keywordResult": {
    "strategy": "KEYWORD",
    "answer": "Mutable structures allow in-place updates; immutable structures return new copies.",
    "confidence": 0.85,
    "evidence": [
      { "source": "internal knowledge", "excerpt": "Mutable data structures permit modification after creation, sharing state across references.", "relevanceScore": 0.9 }
    ],
    "completedAt": "2026-06-28T10:12:03Z"
  },
  "semanticResult": {
    "strategy": "SEMANTIC",
    "answer": "Immutable structures are inherently thread-safe because they cannot be modified after construction.",
    "confidence": 0.78,
    "evidence": [
      { "source": "semantic cluster: concurrency", "excerpt": "Thread safety in functional programming relies on value semantics and immutability.", "relevanceScore": 0.82 }
    ],
    "completedAt": "2026-06-28T10:12:05Z"
  },
  "chainOfThoughtResult": {
    "strategy": "CHAIN_OF_THOUGHT",
    "answer": "Since immutable values cannot change, any two references to the same value are interchangeable, eliminating data races.",
    "confidence": 0.91,
    "evidence": [
      { "source": "reasoning step 1", "excerpt": "Define mutable: a structure whose internal state may change after construction.", "relevanceScore": 0.95 },
      { "source": "reasoning step 2", "excerpt": "If state cannot change, concurrent readers need no synchronization.", "relevanceScore": 0.88 }
    ],
    "completedAt": "2026-06-28T10:12:07Z"
  },
  "answer": {
    "answer": "Mutable data structures allow in-place modification; immutable ones return new copies on change, making them safe for concurrent use.",
    "summary": "All three strategies converged. Keyword search identified the core definition; semantic retrieval added the thread-safety implication; chain-of-thought confirmed both points via a formal derivation. No significant divergence was observed.",
    "strategyResults": [ { "strategy": "KEYWORD" }, { "strategy": "SEMANTIC" }, { "strategy": "CHAIN_OF_THOUGHT" } ],
    "guardrailVerdict": "ok",
    "synthesizedAt": "2026-06-28T10:12:11Z"
  },
  "failureReason": null,
  "agreementScore": 5,
  "agreementRationale": "All three strategies agree in substance; synthesis summary accurately captures convergence.",
  "createdAt": "2026-06-28T10:11:58Z",
  "finishedAt": "2026-06-28T10:12:11Z"
}
```

Lifecycle fields (`keywordResult`, `semanticResult`, `chainOfThoughtResult`, `answer`, `failureReason`, `agreementScore`, `agreementRationale`, `finishedAt`) are `Optional<T>` in Java and serialize as the raw value or `null` — never an `{ "value": … }` wrapper (Lesson 6).

### SSE event format

```
event: query-update
data: { "queryId": "qy-3a1…", "status": "RUNNING", ... }
```

One event per state transition. Clients reconcile by `queryId`. The `AgreementScored` transition emits a `query-update` event whose status is unchanged but whose `agreementScore` is now populated.
