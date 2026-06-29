# API contract — text-to-sql-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ QueryRow... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 QueryResult` / `404` | `QueryEndpoint` ← `QueryView` |
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
  "question": "What is the total amount spent per merchant for the last 30 days?",
  "submittedBy": "analyst-007"
}
```

### QueryRow (list response body — one element)

```json
{
  "queryId": "q-3af...",
  "question": "What is the total amount spent per merchant for the last 30 days?",
  "submittedBy": "analyst-007",
  "submittedAt": "2026-06-28T14:10:00Z",
  "generatedSql": {
    "sql": "SELECT merchant, SUM(amount_cents)/100.0 AS total_amount, currency, COUNT(*) AS receipt_count FROM receipts WHERE receipt_date >= DATE('now','-30 days') GROUP BY merchant, currency ORDER BY total_amount DESC LIMIT 20",
    "explanation": "Aggregates receipt amounts by merchant over the last 30 days, ordered by total descending."
  },
  "sanitized": {
    "rows": [
      { "merchant": "Acme Travel", "total_amount": 4320.00, "currency": "USD", "receipt_count": 12 },
      { "merchant": "QuickBite", "total_amount": 1890.50, "currency": "USD", "receipt_count": 34 }
    ],
    "piiCategoriesFound": []
  },
  "status": "SANITIZED",
  "createdAt": "2026-06-28T14:10:00Z",
  "finishedAt": "2026-06-28T14:10:08Z"
}
```

### QueryResult (single-item response body — full audit view)

```json
{
  "queryId": "q-3af...",
  "request": {
    "queryId": "q-3af...",
    "question": "Which customer had the highest single receipt amount in Q2?",
    "submittedBy": "analyst-007",
    "submittedAt": "2026-06-28T14:11:00Z"
  },
  "generatedSql": {
    "sql": "SELECT customer_name, customer_email, merchant, amount_cents/100.0 AS amount, currency, receipt_date FROM receipts WHERE receipt_date >= '2026-04-01' AND receipt_date < '2026-07-01' ORDER BY amount_cents DESC LIMIT 1",
    "explanation": "Returns the single highest-value receipt in Q2 2026."
  },
  "rawResult": {
    "generatedSql": { "sql": "...", "explanation": "..." },
    "rows": [
      { "customer_name": "Jane Doe", "customer_email": "jane.doe@example.com", "merchant": "CloudSoft Inc", "amount": 1200.00, "currency": "USD", "receipt_date": "2026-05-14" }
    ],
    "summary": "Jane Doe holds the highest single receipt in Q2 at $1,200.00 with CloudSoft Inc on 2026-05-14.",
    "executedAt": "2026-06-28T14:11:06Z"
  },
  "sanitized": {
    "rows": [
      { "customer_name": "[REDACTED-NAME]", "customer_email": "[REDACTED-EMAIL]", "merchant": "CloudSoft Inc", "amount": 1200.00, "currency": "USD", "receipt_date": "2026-05-14" }
    ],
    "piiCategoriesFound": ["customer-name", "email"]
  },
  "status": "SANITIZED",
  "createdAt": "2026-06-28T14:11:00Z",
  "finishedAt": "2026-06-28T14:11:07Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-3af...", "status": "SANITIZED", "sanitized": { "rows": [...], "piiCategoriesFound": ["customer-name"] }, ... }
```

One event per state transition (`SUBMITTED`, `GENERATING`, `EXECUTING`, `EXECUTED`, `SANITIZED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body. Access to `GET /api/queries/{id}` (which returns raw result rows) should be restricted to authorised users; the list and SSE endpoints return only the sanitized form.
