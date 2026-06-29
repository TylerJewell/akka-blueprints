# API contract — ambient-expense-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/expenses` | `SubmitReceiptRequest` | `201 { expenseId }` | `ExpenseEndpoint` → `ExpenseEntity` |
| `GET` | `/api/expenses` | — | `200 [ ExpenseSubmission... ]` (newest-first) | `ExpenseEndpoint` ← `ExpenseView` |
| `GET` | `/api/expenses/{id}` | — | `200 ExpenseSubmission` / `404` | `ExpenseEndpoint` ← `ExpenseView` |
| `GET` | `/api/expenses/sse` | — | `text/event-stream` | `ExpenseEndpoint` ← `ExpenseView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitReceiptRequest (request body)

```json
{
  "rawReceiptText": "THE CAPITAL GRILLE\n123 Main St, Boston MA\nServer: J. Smith\nEntrée: $38.00\nWine: $45.00\nTotal: $83.00\nCard: VISA xxxx-4521",
  "submittedBy": "employee-8821",
  "tripCode": "CONF-2026-Q3"
}
```

`tripCode` is optional. `rawReceiptText` may be a hotel folio, a voice transcript, or any free-text receipt content.

### ExpenseSubmission (response body)

```json
{
  "expenseId": "e-7a2bf...",
  "request": {
    "expenseId": "e-7a2bf...",
    "rawReceiptText": "(raw text preserved for audit)",
    "submittedBy": "employee-8821",
    "tripCode": "CONF-2026-Q3",
    "submittedAt": "2026-06-25T19:00:00Z"
  },
  "sanitized": {
    "redactedText": "THE CAPITAL GRILLE\n[REDACTED-ADDRESS]\nServer: [REDACTED-NAME]\nEntrée: $38.00\nWine: $45.00\nTotal: $83.00\nCard: [REDACTED-CARD]",
    "piiCategoriesFound": ["address", "person-name", "payment-card-number"]
  },
  "report": {
    "expenseId": "e-7a2bf...",
    "lineItems": [
      {
        "lineItemId": "item-1",
        "description": "Dinner - entrée",
        "amount": 38.00,
        "currencyCode": "USD",
        "categoryId": "meals-and-entertainment",
        "merchantName": "The Capital Grille",
        "expenseDate": "2026-06-25",
        "lineStatus": "APPROVED",
        "blockReason": null
      },
      {
        "lineItemId": "item-2",
        "description": "Dinner - wine",
        "amount": 45.00,
        "currencyCode": "USD",
        "categoryId": "meals-and-entertainment",
        "merchantName": "The Capital Grille",
        "expenseDate": "2026-06-25",
        "lineStatus": "APPROVED",
        "blockReason": null
      }
    ],
    "totalAmount": 83.00,
    "currencyCode": "USD",
    "reportStatus": "FULLY_APPROVED",
    "capturedAt": "2026-06-25T19:00:22Z"
  },
  "status": "SUBMITTED_TO_SYSTEM",
  "createdAt": "2026-06-25T19:00:00Z",
  "finishedAt": "2026-06-25T19:00:25Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: expense-update
data: { "expenseId": "e-7a2bf...", "status": "REPORT_READY", "report": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `CAPTURING`, `REPORT_READY`, `SUBMITTED_TO_SYSTEM`, `FAILED`). Clients reconcile by `expenseId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body. `tripCode` should be validated against the deployer's project code registry.
