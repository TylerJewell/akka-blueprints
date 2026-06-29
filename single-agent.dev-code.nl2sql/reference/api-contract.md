# API contract — nl2sql

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
  "question": "What is total revenue per product this month?",
  "schemaContext": "orders",
  "submittedBy": "analyst-42"
}
```

### Query (response body)

```json
{
  "queryId": "q-9ac...",
  "request": {
    "queryId": "q-9ac...",
    "question": "What is total revenue per product this month?",
    "schemaContext": "orders",
    "submittedBy": "analyst-42",
    "submittedAt": "2026-06-28T09:15:00Z"
  },
  "schema": {
    "contextName": "orders",
    "tables": [
      {
        "tableName": "orders",
        "description": "One row per customer order",
        "columns": [
          { "columnName": "order_id", "dataType": "uuid", "nullable": false },
          { "columnName": "customer_id", "dataType": "uuid", "nullable": false },
          { "columnName": "created_at", "dataType": "timestamp", "nullable": false },
          { "columnName": "total_amount", "dataType": "numeric", "nullable": false }
        ]
      },
      {
        "tableName": "order_items",
        "description": "Line items within an order",
        "columns": [
          { "columnName": "item_id", "dataType": "uuid", "nullable": false },
          { "columnName": "order_id", "dataType": "uuid", "nullable": false },
          { "columnName": "product_id", "dataType": "uuid", "nullable": false },
          { "columnName": "quantity", "dataType": "integer", "nullable": false },
          { "columnName": "unit_price", "dataType": "numeric", "nullable": false }
        ]
      },
      {
        "tableName": "products",
        "description": "Product catalogue",
        "columns": [
          { "columnName": "product_id", "dataType": "uuid", "nullable": false },
          { "columnName": "name", "dataType": "text", "nullable": false },
          { "columnName": "category", "dataType": "text", "nullable": true }
        ]
      }
    ]
  },
  "result": {
    "generatedSql": "SELECT p.name, SUM(oi.quantity * oi.unit_price) AS revenue FROM products p INNER JOIN order_items oi ON oi.product_id = p.product_id INNER JOIN orders o ON o.order_id = oi.order_id WHERE o.created_at >= date_trunc('month', NOW()) GROUP BY p.name ORDER BY revenue DESC LIMIT 1000",
    "columnNames": ["name", "revenue"],
    "rows": [
      ["Widget A", 4820.50],
      ["Widget B", 3110.00],
      ["Widget C", 1250.75]
    ],
    "rowCount": 3,
    "summary": "Three products generated revenue this month; Widget A leads with $4,820.50.",
    "executedAt": "2026-06-28T09:15:12Z"
  },
  "score": {
    "score": 5,
    "rationale": "Query returns named columns with a LIMIT; result set is non-empty and covers expected schema columns.",
    "scoredAt": "2026-06-28T09:15:12Z"
  },
  "status": "SCORED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:12Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### Query (HALTED example)

```json
{
  "queryId": "q-7ff...",
  "request": { "question": "Delete all old orders", "schemaContext": "orders", "submittedBy": "analyst-01", "submittedAt": "2026-06-28T09:20:00Z" },
  "schema": { ... },
  "result": null,
  "score": null,
  "status": "HALTED",
  "haltReason": "Unsafe SQL detected: keyword DELETE at position 0. Task terminated.",
  "createdAt": "2026-06-28T09:20:00Z",
  "finishedAt": "2026-06-28T09:20:03Z"
}
```

### SSE event format

```
event: query-update
data: { "queryId": "q-9ac...", "status": "SCORED", "result": { ... }, "score": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SCHEMA_ATTACHED`, `TRANSLATING`, `RESULT_READY`, `SCORED`, `HALTED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
