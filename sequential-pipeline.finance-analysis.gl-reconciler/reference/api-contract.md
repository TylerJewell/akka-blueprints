# API contract — gl-reconciler

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/reconciliations` | `SubmitRunRequest` | `201 { runId }` | `ReconciliationEndpoint` → `LedgerReconciliationEntity` |
| `GET` | `/api/reconciliations` | — | `200 [ ReconciliationRun... ]` (newest-first) | `ReconciliationEndpoint` ← `ReconciliationView` |
| `GET` | `/api/reconciliations/{id}` | — | `200 ReconciliationRun` / `404` | `ReconciliationEndpoint` ← `ReconciliationView` |
| `GET` | `/api/reconciliations/sse` | — | `text/event-stream` | `ReconciliationEndpoint` ← `ReconciliationView` |
| `POST` | `/api/reconciliations/{id}/approve` | `ApproveRequest` | `200 { runId, status }` | `ReconciliationEndpoint` → `LedgerReconciliationEntity` |
| `POST` | `/api/reconciliations/{id}/reject` | `RejectRequest` | `200 { runId, status }` | `ReconciliationEndpoint` → `LedgerReconciliationEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRunRequest (request body)

```json
{
  "accountSetId": "fund-a-q2-close"
}
```

### ApproveRequest (request body)

```json
{
  "reviewedBy": "j.smith@example.com"
}
```

### RejectRequest (request body)

```json
{
  "reviewedBy": "j.smith@example.com",
  "reason": "Variance in account 1100 requires further investigation before posting."
}
```

### ReconciliationRun (response body)

```json
{
  "runId": "r-3bc7e1...",
  "accountSetId": "fund-a-q2-close",
  "snapshot": {
    "accountSetId": "fund-a-q2-close",
    "entries": [
      {
        "entryId": "e-001",
        "accountId": "1100",
        "description": "Cash receipt — June settlement",
        "debit": "125000.00",
        "credit": "0.00",
        "postingDate": "2026-06-30"
      }
    ],
    "expectedBalances": [
      { "accountId": "1100", "accountName": "Cash and equivalents", "expectedBalance": "125000.00", "currency": "USD" }
    ],
    "fetchedAt": "2026-06-29T08:00:00Z"
  },
  "reconciliation": {
    "variances": [
      { "accountId": "1100", "expectedBalance": "125000.00", "actualBalance": "125000.00", "delta": "0.00", "currency": "USD", "isMaterial": false }
    ],
    "nav": {
      "fundId": "fund-a",
      "totalAssets": "125000.00",
      "totalLiabilities": "4200.00",
      "nav": "120800.00",
      "calculatedAt": "2026-06-29T08:00:05Z"
    },
    "allAccountsReconciled": true,
    "hasMaterialVariance": false,
    "reconciledAt": "2026-06-29T08:00:05Z"
  },
  "escalation": null,
  "journal": {
    "journalId": "jnl-fund-a-q2-close-20260629",
    "lines": [],
    "totalDebits": "0.00",
    "totalCredits": "0.00",
    "periodEnd": "2026-06-30",
    "draftedAt": "2026-06-29T08:00:10Z"
  },
  "validation": {
    "passed": true,
    "findings": [],
    "validatedAt": "2026-06-29T08:00:10Z"
  },
  "status": "POSTED",
  "createdAt": "2026-06-29T08:00:00Z",
  "finishedAt": "2026-06-29T08:00:11Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). The `escalation` field is `null` on the happy path; when a material variance was detected it carries the full `EscalationRecord` including `reviewedBy`, `decision`, and `decidedAt`.

### ReconciliationRun — PENDING_APPROVAL state example

```json
{
  "runId": "r-9fa2d0...",
  "accountSetId": "multi-currency-sweep-june",
  "snapshot": { "..." : "..." },
  "reconciliation": {
    "variances": [
      { "accountId": "2300", "expectedBalance": "50000.00", "actualBalance": "52300.00", "delta": "2300.00", "currency": "EUR", "isMaterial": true }
    ],
    "nav": null,
    "allAccountsReconciled": false,
    "hasMaterialVariance": true,
    "reconciledAt": "2026-06-29T09:15:00Z"
  },
  "escalation": {
    "escalationId": "esc-r-9fa2d0",
    "runId": "r-9fa2d0...",
    "reason": "Material variance in account 2300: delta EUR 2300.00 exceeds 1% threshold (EUR 500.00).",
    "reviewedBy": null,
    "decision": null,
    "raisedAt": "2026-06-29T09:15:01Z",
    "decidedAt": null
  },
  "journal": null,
  "validation": null,
  "status": "PENDING_APPROVAL",
  "createdAt": "2026-06-29T09:14:00Z",
  "finishedAt": null
}
```

### SSE event format

```
event: run-update
data: { "runId": "r-3bc7e1...", "status": "RECONCILED", "reconciliation": { ... }, ... }
```

One event per state transition (`CREATED`, `FETCHING`, `FETCHED`, `RECONCILING`, `RECONCILED`, `PENDING_APPROVAL`, `DRAFTING`, `DRAFT_VALIDATED`, `POSTED`, `REJECTED`, `FAILED`) and one per `ValidationFailed` audit event:

```
event: run-validation-failed
data: { "runId": "r-3bc7e1...", "findings": ["balance-rule: totalDebits 500.00 != totalCredits 450.00"], "failedAt": "..." }
```

Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp `submittedBy` and `approvedBy` fields from the authenticated principal — extend `SubmitRunRequest`, `RunCreated`, `ApproveRequest`, and `EscalationApproved` to carry the identity, and enforce that only users in the `finance-controller` role may call `/approve` and `/reject`.
