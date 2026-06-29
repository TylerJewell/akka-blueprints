# API contract — text-to-sql-guarded

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ QueryRecord... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 QueryRecord` / `404` | `QueryEndpoint` ← `QueryView` |
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
  "question": "What were the top 10 vendors by spend last quarter?"
}
```

### QueryRecord (response body)

```json
{
  "queryId": "q-3bc7f1...",
  "question": "What were the top 10 vendors by spend last quarter?",
  "parsedQuery": {
    "question": "What were the top 10 vendors by spend last quarter?",
    "sql": "SELECT vendor_name, SUM(amount) AS total_spend FROM transactions WHERE transaction_date >= DATE_SUB(CURRENT_DATE, INTERVAL 1 QUARTER) GROUP BY vendor_name ORDER BY total_spend DESC LIMIT 10",
    "tablesReferenced": [
      {
        "tableName": "transactions",
        "columnNames": ["transaction_id", "vendor_name", "amount", "transaction_date", "account_number"],
        "description": "Financial transaction ledger"
      }
    ],
    "parsedAt": "2026-06-28T10:00:00Z"
  },
  "rawResult": null,
  "report": {
    "question": "What were the top 10 vendors by spend last quarter?",
    "sqlUsed": "SELECT vendor_name, SUM(amount) AS total_spend FROM transactions ...",
    "narrative": "Acme Supplies led vendor spend last quarter at $142,000, followed by Globe Services at $98,500.",
    "result": {
      "columns": ["vendor_name", "total_spend"],
      "rows": [
        { "cells": { "vendor_name": "Acme Supplies", "total_spend": "142000.00" } },
        { "cells": { "vendor_name": "Globe Services", "total_spend": "98500.00" } }
      ],
      "redactionSummary": { "redactionCount": 0, "redactedFields": [] }
    },
    "formattedAt": "2026-06-28T10:00:12Z"
  },
  "haltReason": null,
  "status": "FORMATTED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:12Z",
  "guardrailRejections": []
}
```

A halted query:

```json
{
  "queryId": "q-8da2c9...",
  "question": "Delete all transactions from last year",
  "parsedQuery": {
    "sql": "DELETE FROM transactions WHERE YEAR(transaction_date) = 2025",
    "tablesReferenced": [],
    "parsedAt": "2026-06-28T10:01:00Z"
  },
  "rawResult": null,
  "report": null,
  "haltReason": "Destructive SQL pattern detected: DELETE. Statement blocked before database contact.",
  "status": "HALTED",
  "createdAt": "2026-06-28T10:01:00Z",
  "finishedAt": "2026-06-28T10:01:02Z",
  "guardrailRejections": []
}
```

A query with a guardrail rejection:

```json
{
  "queryId": "q-5ef0a3...",
  "question": "Show me accounts payable aging over 90 days",
  "status": "FORMATTED",
  "guardrailRejections": [
    {
      "phase": "PARSE",
      "tool": "executeSql",
      "reason": "phase-violation: executeSql requires status in {PARSED, QUERYING} with parsedQuery present, saw PARSING",
      "rejectedAt": "2026-06-28T10:02:01Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `rawResult` is stored on the entity for the audit trail but is **never** included in the public API response — the API returns only the sanitized `report.result`.

### SSE event format

```
event: query-update
data: { "queryId": "q-3bc7f1...", "status": "QUERIED", "parsedQuery": { ... }, ... }
```

One event per state transition (`CREATED`, `PARSING`, `PARSED`, `QUERYING`, `QUERIED`, `FORMATTING`, `FORMATTED`, `HALTED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: query-rejection
data: { "queryId": "q-5ef0a3...", "phase": "PARSE", "tool": "executeSql", "reason": "...", "rejectedAt": "..." }
```

And one per `SafetyHaltFired`:

```
event: query-halted
data: { "queryId": "q-8da2c9...", "matchedPattern": "DELETE", "sql": "DELETE FROM ...", "haltedAt": "..." }
```

Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitQueryRequest` record and the `QueryCreated` event to carry it.
