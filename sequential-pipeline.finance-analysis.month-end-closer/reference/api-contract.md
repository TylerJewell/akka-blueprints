# API contract — month-end-closer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/close-runs` | `StartCloseRunRequest` | `201 { closeRunId }` | `CloseRunEndpoint` → `CloseRunEntity` |
| `GET` | `/api/close-runs` | — | `200 [ CloseRunRecord... ]` (newest-first) | `CloseRunEndpoint` ← `CloseRunView` |
| `GET` | `/api/close-runs/{id}` | — | `200 CloseRunRecord` / `404` | `CloseRunEndpoint` ← `CloseRunView` |
| `POST` | `/api/close-runs/{id}/approve` | `ApprovalRequest` | `204` | `CloseRunEndpoint` → `CloseRunEntity` |
| `POST` | `/api/close-runs/{id}/reject` | `ApprovalRequest` | `204` | `CloseRunEndpoint` → `CloseRunEntity` |
| `GET` | `/api/close-runs/sse` | — | `text/event-stream` | `CloseRunEndpoint` ← `CloseRunView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartCloseRunRequest (request body)

```json
{
  "entity": "ACME-US",
  "period": "2026-05"
}
```

### ApprovalRequest (request body for approve and reject)

```json
{
  "step": "GATHER",
  "approver": "jane.smith@acme.com",
  "comment": "Ledger lines look complete. Proceeding to validate."
}
```

`step` is `"GATHER"` or `"VALIDATE"`. `comment` is optional (may be `null`).

### CloseRunRecord (response body)

```json
{
  "closeRunId": "cr-7f3a1b22",
  "entity": "ACME-US",
  "period": "2026-05",
  "ledgerSnapshot": {
    "entity": "ACME-US",
    "period": "2026-05",
    "lines": [
      {
        "accountCode": "6100",
        "description": "Marketing accrual May 2026",
        "debitAmount": 15000.00,
        "creditAmount": 0.00,
        "postingDate": "2026-05-31"
      }
    ],
    "chartOfAccounts": {
      "accounts": [
        { "code": "6100", "name": "Marketing Expense", "type": "EXPENSE" },
        { "code": "2010", "name": "Accrued Liabilities", "type": "LIABILITY" }
      ]
    },
    "gatheredAt": "2026-06-29T09:00:00Z"
  },
  "journalEntrySet": {
    "entries": [
      {
        "entryId": "je-001",
        "debitAccount": "6100",
        "creditAccount": "2010",
        "amount": 15000.00,
        "narration": "Marketing accrual May 2026",
        "reversalDate": "2026-06-01"
      }
    ],
    "validatedAt": "2026-06-29T09:01:00Z"
  },
  "closePackage": {
    "title": "ACME-US Month-End Close — 2026-05",
    "period": "2026-05",
    "entity": "ACME-US",
    "trialBalance": {
      "totalDebits": 15000.00,
      "totalCredits": 15000.00,
      "variance": 0.00,
      "balanced": true
    },
    "journalEntries": [
      {
        "entryId": "je-001",
        "debitAccount": "6100",
        "creditAccount": "2010",
        "amount": 15000.00,
        "narration": "Marketing accrual May 2026",
        "reversalDate": "2026-06-01"
      }
    ],
    "varianceCommentary": "All entries balance. Marketing accrual of $15,000 recorded against accrued liabilities with reversal on 2026-06-01.",
    "writtenAt": "2026-06-29T09:02:00Z"
  },
  "reconciliation": {
    "score": 5,
    "rationale": "Debit/credit balance, account-code validity, accrual reversal completeness, and variance threshold all satisfied.",
    "evaluatedAt": "2026-06-29T09:02:05Z"
  },
  "approvals": [
    {
      "step": "GATHER",
      "approver": "jane.smith@acme.com",
      "decision": "APPROVED",
      "comment": "Ledger lines look complete. Proceeding to validate.",
      "decidedAt": "2026-06-29T09:00:45Z"
    },
    {
      "step": "VALIDATE",
      "approver": "jane.smith@acme.com",
      "decision": "APPROVED",
      "comment": null,
      "decidedAt": "2026-06-29T09:01:30Z"
    }
  ],
  "status": "EVALUATED",
  "createdAt": "2026-06-29T08:59:55Z",
  "finishedAt": "2026-06-29T09:02:05Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `approvals` is an array — empty on a freshly created run, populated as the run progresses.

### SSE event format

```
event: close-run-update
data: { "closeRunId": "cr-7f3a1b22", "status": "AWAITING_GATHER_APPROVAL", "ledgerSnapshot": { ... }, ... }
```

One event per state transition (`CREATED`, `GATHERING`, `AWAITING_GATHER_APPROVAL`, `VALIDATING`, `AWAITING_VALIDATE_APPROVAL`, `REPORTING`, `EVALUATED`, `FAILED`) and one per approval/rejection:

```
event: close-run-approval
data: { "closeRunId": "cr-7f3a1b22", "step": "GATHER", "approver": "jane.smith@acme.com", "decision": "APPROVED", "decidedAt": "..." }
```

Clients reconcile by `closeRunId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp `submittedBy` and `approver` fields from the authenticated principal — extend `StartCloseRunRequest`, `ApprovalRequest`, `CloseRunCreated`, `StepApproved`, and `StepRejected` to carry the authenticated identity rather than accepting it from the request body.
