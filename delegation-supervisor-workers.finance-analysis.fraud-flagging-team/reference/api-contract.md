# API contract — fraud-flagging-team

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Served by `FraudEndpoint` (`/api/*`) and `AppEndpoint` (`/`, `/app/*`).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/cases` | `TransactionInput` | `{ "caseId": "uuid" }` | FraudEndpoint → CaseEntity, TransactionQueue |
| POST | `/api/cases/{caseId}/confirm` | `{ "confirmedBy": "string" }` | `200` \| `404` | FraudEndpoint → CaseEntity |
| POST | `/api/cases/{caseId}/dismiss` | `{ "dismissedBy": "string", "note": "string" }` | `200` \| `404` | FraudEndpoint → CaseEntity |
| GET | `/api/cases?status=...` | — | `{ "cases": [FraudCase, ...] }` | FraudEndpoint → CasesView (filter client-side) |
| GET | `/api/cases/{caseId}` | — | `FraudCase` | FraudEndpoint → CaseEntity |
| GET | `/api/cases/sse` | — | SSE of `FraudCase` | FraudEndpoint → CasesView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | FraudEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | FraudEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | FraudEndpoint |
| GET | `/` | — | `302 → /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

The `?status=` filter is applied client-side in the endpoint because the view has no enum WHERE clause (Lesson 2).

## Payload shapes

`TransactionInput`:

```json
{ "customerId": "C-4821", "amount": 9400.00, "transactionRef": "TXN-55190", "memo": "wire to new payee" }
```

`FraudCase` (lifecycle fields are nullable — `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "customerId": "C-4821",
  "transactionRef": "TXN-55190 or null",
  "amount": 9400.00,
  "status": "ANALYZING | FLAGGED | CLEARED | CONFIRMED | ACTIONED | DISMISSED | ESCALATED",
  "openedAt": "ISO-8601 or null",
  "fraudScore": "0.0-1.0 or null",
  "fraudSignals": "string or null",
  "compliant": "boolean or null",
  "complianceNotes": "string or null",
  "riskTier": "LOW | MEDIUM | HIGH or null",
  "riskRationale": "string or null",
  "supervisorRationale": "string or null",
  "verdictConfidence": "0.0-1.0 or null",
  "flaggedAt": "ISO-8601 or null",
  "clearedAt": "ISO-8601 or null",
  "confirmedAt": "ISO-8601 or null",
  "confirmedBy": "string or null",
  "analystNote": "string or null",
  "dismissedAt": "ISO-8601 or null",
  "escalatedAt": "ISO-8601 or null",
  "actionedAt": "ISO-8601 or null",
  "actionTaken": "string or null"
}
```

## SSE event format

`GET /api/cases/sse` streams the view rows as they change. Each event:

```
event: case
data: { ...FraudCase JSON as above... }
```

The UI subscribes via `EventSource` and upserts each case into the live list by `id`.
