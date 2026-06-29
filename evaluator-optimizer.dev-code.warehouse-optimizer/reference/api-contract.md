# API contract — warehouse-optimizer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/requests` | `{ "originalSql": String, "objective"?: String, "submittedBy"?: String }` | `202 { "requestId": String }` | `OptimizerEndpoint` → `RequestQueue` |
| `GET` | `/api/requests` | — | `200 [ OptimizationRequest... ]` (optional `?status=PROPOSING\|AWAITING_DBA\|EVALUATING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllRequests`) | `OptimizerEndpoint` ← `RequestsView` |
| `GET` | `/api/requests/{id}` | — | `200 OptimizationRequest` / `404` | `OptimizerEndpoint` ← `RequestsView` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` (one event per request change) | `OptimizerEndpoint` ← `RequestsView` |
| `POST` | `/api/requests/{id}/dba-decision` | `{ "approved": Boolean, "decisionNote"?: String, "decidedBy"?: String }` | `200` / `404` / `409` (request not in AWAITING_DBA) | `OptimizerEndpoint` → `OptimizationRequestEntity` → `OptimizationWorkflow` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/requests`

- Missing `objective` → `"improve performance"`.
- Missing `submittedBy` → `"anonymous"`.
- `originalSql` must be non-empty; otherwise `400`.
- Duplicate-detection window: 10 s on `(originalSql, submittedBy)`; the second submission returns the first `requestId` (200) instead of starting a new workflow.

## Defaults on `POST /api/requests/{id}/dba-decision`

- Missing `decisionNote` → `""`.
- Missing `decidedBy` → `"dba"`.
- Returns `409` if the request is not currently in `AWAITING_DBA`.

## JSON shapes

### OptimizationRequest

```json
{
  "requestId": "wr-9a1b2c…",
  "originalSql": "SELECT * FROM orders WHERE customer_id = 42",
  "objective": "reduce full-table scan on orders",
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "proposal": {
        "proposedSql": "SELECT id, total FROM orders WHERE customer_id = 42 AND created_at >= '2024-01-01'",
        "kind": "QUERY_REWRITE",
        "rationale": "Added date-range predicate on created_at to enable index range scan.",
        "proposedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "dbaDecision": null,
      "evaluation": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "SELECT list not covered by the index — heap reads remain.",
            "Date-range selectivity not quantified in rationale.",
            "ORDER BY clause not addressed."
          ],
          "overallRationale": "Objective alignment is reasonable but the index definition is incomplete."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "proposal": {
        "proposedSql": "SELECT id, total FROM orders WHERE customer_id = 42 AND created_at >= '2024-01-01'",
        "kind": "INDEX_ADD",
        "rationale": "Proposes covering index on (customer_id, created_at) INCLUDE (id, total) to eliminate heap reads and cover the projection.",
        "proposedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "dbaDecision": null,
      "evaluation": {
        "verdict": "APPROVE",
        "notes": { "bullets": [], "overallRationale": "Covering index addresses all four rubric dimensions." },
        "score": 5,
        "evaluatedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedSql": "SELECT id, total FROM orders WHERE customer_id = 42 AND created_at >= '2024-01-01'",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",           "detail": "" }
{ "passed": false, "reasonCode": "DDL_DETECTED",  "detail": "ALTER keyword detected in proposal; routed to DBA approval gate." }
```

`reasonCode` is one of `OK`, `DDL_DETECTED`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### DBA decision form

```json
{ "approved": true,  "decisionNote": "Index addition reviewed; safe to proceed.", "decidedBy": "j.smith", "decidedAt": "2026-06-28T09:05:00Z" }
{ "approved": false, "decisionNote": "Column referenced by downstream ETL — do not drop.", "decidedBy": "j.smith", "decidedAt": "2026-06-28T09:05:10Z" }
```

### SSE event format

```
event: request-update
data: { "requestId": "wr-9a1b2c…", "status": "EVALUATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `requestId`. The full `OptimizationRequest` JSON is included so a fresh client can render the row without a separate fetch.
