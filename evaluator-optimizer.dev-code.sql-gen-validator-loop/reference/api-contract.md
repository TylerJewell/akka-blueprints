# API contract — sql-gen-validator-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `{ "question": String, "schemaName"?: String, "requestedBy"?: String }` | `202 { "requestId": String }` | `SqlEndpoint` → `RequestQueue` |
| `GET` | `/api/queries` | — | `200 [ QueryRequest... ]` (optional `?status=GENERATING\|VALIDATING\|ACCEPTED\|FAILED_FINAL` — filtered client-side from `getAllRequests`) | `SqlEndpoint` ← `QueryRequestView` |
| `GET` | `/api/queries/{id}` | — | `200 QueryRequest` / `404` | `SqlEndpoint` ← `QueryRequestView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` (one event per request change) | `SqlEndpoint` ← `QueryRequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/queries`

- Missing `schemaName` → `"default_schema"` (from `sql-gen.workflow.default-schema`).
- Missing `requestedBy` → `"anonymous"`.
- `question` must be non-empty; otherwise `400`.
- Duplicate-detection window: 10 s on `(question, schemaName, requestedBy)`; the second submission returns the first `requestId` (200) instead of starting a new workflow.

## JSON shapes

### QueryRequest

```json
{
  "requestId": "qr-9a1f2…",
  "question": "Show me the top 10 customers by total order value",
  "schemaName": "orders_schema",
  "maxAttempts": 4,
  "status": "ACCEPTED",
  "attempts": [
    {
      "attemptNumber": 1,
      "query": {
        "sql": "SELECT c.id, c.name, SUM(o.total) AS total_order_value FROM customers c JOIN orders o ON o.customer_id = c.id GROUP BY c.id, c.name ORDER BY total_order_value DESC LIMIT 10",
        "dialect": "standard-sql",
        "generatedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "validation": {
        "verdict": "INVALID",
        "notes": {
          "bullets": [
            "Column 'total' does not exist in table 'orders'; the correct column is 'order_total'.",
            "Missing index hint on large orders table; consider adding WHERE clause.",
            "Ambiguous column reference 'id' — qualify as 'customers.id'."
          ],
          "overallRationale": "Schema correctness falls below threshold due to wrong column name."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "query": {
        "sql": "SELECT c.id, c.name, SUM(o.order_total) AS total_order_value FROM customers c JOIN orders o ON o.customer_id = c.id GROUP BY c.id, c.name ORDER BY total_order_value DESC LIMIT 10",
        "dialect": "standard-sql",
        "generatedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "validation": {
        "verdict": "VALID",
        "notes": { "bullets": [], "overallRationale": "All tables and columns resolve; join condition is sound; question coverage is complete." },
        "score": 5,
        "evaluatedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "acceptedAttemptNumber": 2,
  "acceptedSql": "SELECT c.id, c.name, SUM(o.order_total) AS total_order_value FROM customers c JOIN orders o ON o.customer_id = c.id GROUP BY c.id, c.name ORDER BY total_order_value DESC LIMIT 10",
  "failureReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",                 "detail": "" }
{ "passed": false, "reasonCode": "MUTATION_FORBIDDEN",  "detail": "Query contains keyword 'DELETE'; only SELECT is permitted." }
```

`reasonCode` is one of `OK`, `MUTATION_FORBIDDEN`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: query-update
data: { "requestId": "qr-9a1f2…", "status": "VALIDATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `requestId`. The full `QueryRequest` JSON is included so a fresh client can render the row without a separate fetch.
