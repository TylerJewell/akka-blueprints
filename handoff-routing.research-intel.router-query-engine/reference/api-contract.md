# API contract — router-query-engine

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first); supports optional `?engineType=…&status=…` filtered client-side | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `QueryView` |
| `POST` | `/api/queries` | `{ "questionText": String, "source": String }` | `201 { "queryId": String }` | `QueryEndpoint` → `QuestionQueue` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### ResearchQuestion (request body for `POST /api/queries`, minus `queryId` and `receivedAt`)

```json
{
  "questionText": "How many transactions had status='failed' in Q1 2026?",
  "source": "manual"
}
```

### Query

```json
{
  "queryId": "q-3de91a47",
  "question": {
    "queryId": "q-3de91a47",
    "questionText": "How many transactions had status='failed' in Q1 2026?",
    "source": "simulator",
    "receivedAt": "2026-06-28T09:00:00Z"
  },
  "routing": {
    "engineType": "STRUCTURED",
    "confidence": "high",
    "reason": "Explicit count filter on a named status field with a date range."
  },
  "answer": {
    "answerText": "4,218 transactions had status='failed' in Q1 2026.",
    "action": "AGGREGATION_RESULT",
    "engineTag": "structured",
    "sourceRefs": ["transactions_table"],
    "answeredAt": "2026-06-28T09:00:14Z"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Engine type right; reason names the count filter and date range directly.",
    "scoredAt": "2026-06-28T09:00:10Z"
  },
  "escalationReason": null,
  "status": "ANSWERED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:14Z"
}
```

An escalated query has `routing.engineType = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`, `finishedAt` set.

### RoutingDecision (returned by `RouterAgent`, embedded in `Query.routing`)

```json
{ "engineType": "STRUCTURED" | "SEMANTIC" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### Answer (returned by either engine, embedded in `Query.answer`)

```json
{ "answerText": "4,218 transactions had status='failed' in Q1 2026.",
  "action": "DIRECT_LOOKUP" | "AGGREGATION_RESULT" | "CONCEPT_SUMMARY" | "SYNTHESIS" | "ESCALATED",
  "engineTag": "structured" | "semantic",
  "sourceRefs": ["transactions_table"],
  "answeredAt": "ISO-8601" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `Query.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: query-update
data: { "queryId": "q-…", "status": "ROUTED_STRUCTURED", … full Query JSON … }
```

One event per state transition on the `QueryView`. Clients reconcile by `queryId`.

## Status codes

- `200` — successful read.
- `201` — successful create (`POST /api/queries`).
- `404` — unknown `queryId`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
