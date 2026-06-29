# API contract — rag-baseline

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
  "questionText": "What is the difference between event sourcing and traditional CRUD?",
  "submittedBy": "developer-01"
}
```

### Query (response body)

```json
{
  "queryId": "q-7ca...",
  "request": {
    "queryId": "q-7ca...",
    "questionText": "What is the difference between event sourcing and traditional CRUD?",
    "submittedBy": "developer-01",
    "submittedAt": "2026-06-28T09:15:00Z"
  },
  "retrieved": {
    "passages": [
      {
        "passageId": "es-patterns-01",
        "documentTitle": "Event Sourcing Patterns",
        "text": "Event sourcing records every state change as an immutable event appended to a log.",
        "similarityScore": 0.87
      },
      {
        "passageId": "es-patterns-02",
        "documentTitle": "Event Sourcing Patterns",
        "text": "Current state is computed by replaying all events from the beginning of the log.",
        "similarityScore": 0.81
      }
    ],
    "topK": 5
  },
  "answer": {
    "answerText": "Event sourcing stores state as a sequence of immutable events rather than overwriting a current-state record. The current state is derived by replaying the event log from the beginning.",
    "citations": [
      {
        "passageId": "es-patterns-01",
        "passageExcerpt": "Event sourcing records every state change as an immutable event appended to a log.",
        "claimSupported": "stores state as a sequence of immutable events"
      },
      {
        "passageId": "es-patterns-02",
        "passageExcerpt": "Current state is computed by replaying all events from the beginning of the log.",
        "claimSupported": "current state is derived by replaying the event log"
      }
    ],
    "usedPassages": 2,
    "answeredAt": "2026-06-28T09:15:22Z"
  },
  "eval": {
    "score": 4,
    "rationale": "Both cited passage ids are valid; all checks pass; 2 distinct passages cited.",
    "evaluatedAt": "2026-06-28T09:15:22Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-7ca...", "status": "ANSWER_RECORDED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `PASSAGES_RETRIEVED`, `ANSWERING`, `ANSWER_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
