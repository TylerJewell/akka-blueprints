# API contract — personal-finance-agent

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
  "queryText": "How much did I spend on groceries and dining last month?",
  "accountSetId": "individual-checking",
  "principal": "user-4491"
}
```

`accountSetId` selects the seeded account set. The endpoint resolves the matching `AccountRef` list and `TransactionRecord` list from the in-process simulation and attaches them to the `QueryRequest`.

### Query (response body)

```json
{
  "queryId": "q-9a3...",
  "request": {
    "queryId": "q-9a3...",
    "queryText": "How much did I spend on groceries and dining last month?",
    "accountSetId": "individual-checking",
    "rawTransactions": "(raw list preserved for audit — not shown in view row)",
    "accounts": [
      { "accountId": "acct-001", "label": "Main Checking", "type": "CHECKING" },
      { "accountId": "acct-002", "label": "Savings",       "type": "SAVINGS" }
    ],
    "principal": "user-4491",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "sanitized": {
    "accounts": [
      { "accountId": "[ACCT-001]", "label": "Main Checking", "type": "CHECKING" },
      { "accountId": "[ACCT-002]", "label": "Savings",       "type": "SAVINGS" }
    ],
    "transactions": [
      {
        "transactionId": "txn-881",
        "accountId": "[ACCT-001]",
        "description": "WHOLEFOOD MKT [NAME-INITIALS]",
        "merchant": "Whole Foods",
        "category": "Groceries",
        "amount": 47.30,
        "currency": "USD",
        "date": "2026-05-14"
      }
    ],
    "piiTokensReplaced": 6
  },
  "response": {
    "answer": "You spent $312.50 across 14 transactions in Groceries and Dining during May 2026.",
    "structuredData": "[{\"category\":\"Groceries\",\"total\":187.20,\"count\":9},{\"category\":\"Dining\",\"total\":125.30,\"count\":5}]",
    "toolTrace": [
      {
        "toolName": "listTransactions",
        "arguments": "{\"accountId\":\"[ACCT-001]\",\"fromDate\":\"2026-05-01\",\"toDate\":\"2026-05-31\"}",
        "result": "[...]",
        "outcome": "NOT_A_WRITE",
        "calledAt": "2026-06-28T09:00:12Z"
      },
      {
        "toolName": "groupByCategory",
        "arguments": "{\"transactions\":[...]}",
        "result": "[{\"category\":\"Groceries\",\"total\":187.20,\"count\":9},{\"category\":\"Dining\",\"total\":125.30,\"count\":5}]",
        "outcome": "NOT_A_WRITE",
        "calledAt": "2026-06-28T09:00:13Z"
      }
    ],
    "answeredAt": "2026-06-28T09:00:14Z"
  },
  "status": "ANSWERED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:14Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### Query (blocked-write example, partial)

```json
{
  "queryId": "q-7c1...",
  "response": {
    "answer": "The transfer was blocked. Your Main Checking account has an available balance of $1,240.00, which is less than the requested $5,000.00. No funds were moved.",
    "structuredData": null,
    "toolTrace": [
      {
        "toolName": "getBalance",
        "arguments": "{\"accountId\":\"[ACCT-001]\"}",
        "result": "{\"accountId\":\"[ACCT-001]\",\"available\":1240.00,\"currency\":\"USD\"}",
        "outcome": "NOT_A_WRITE",
        "calledAt": "2026-06-28T10:05:02Z"
      },
      {
        "toolName": "transferFunds",
        "arguments": "{\"fromAccountId\":\"[ACCT-001]\",\"toAccountId\":\"[ACCT-002]\",\"amount\":5000.00,\"note\":\"savings top-up\"}",
        "result": "blocked: amount 5000.00 exceeds available balance 1240.00",
        "outcome": "BLOCKED",
        "calledAt": "2026-06-28T10:05:03Z"
      }
    ],
    "answeredAt": "2026-06-28T10:05:04Z"
  },
  "status": "ANSWERED"
}
```

### SSE event format

```
event: query-update
data: { "queryId": "q-9a3...", "status": "ANSWERED", "response": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `ANSWERING`, `ANSWERED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition so a late-joining client never needs to replay history.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `principal` from the authenticated session rather than the request body. The `WriteGuardrail` checks `principal` against the session subject; this check is a no-op in dev mode unless both fields are non-empty.
