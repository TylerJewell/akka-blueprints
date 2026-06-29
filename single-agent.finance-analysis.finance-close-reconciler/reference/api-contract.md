# API contract — finance-close-reconciler

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/periods` | `SubmitPeriodRequest` | `201 { periodId }` | `ReconciliationEndpoint` → `PeriodEntity` |
| `GET` | `/api/periods` | — | `200 [ Period... ]` (newest-first) | `ReconciliationEndpoint` ← `ReconciliationView` |
| `GET` | `/api/periods/{id}` | — | `200 Period` / `404` | `ReconciliationEndpoint` ← `ReconciliationView` |
| `GET` | `/api/periods/sse` | — | `text/event-stream` | `ReconciliationEndpoint` ← `ReconciliationView` |
| `POST` | `/api/periods/{id}/signoff` | `SignOffRequest` | `204` / `409` if already decided | `ReconciliationEndpoint` → `PeriodEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPeriodRequest (request body)

```json
{
  "periodLabel": "Q2 FY2026",
  "rawBalance": "account_code,description,debit,credit\n1000,Cash,125000.00,0.00\n...",
  "rules": [
    {
      "ruleId": "cash-ar-offset",
      "accountPair": "1000/1200",
      "description": "Cash account must offset AR balance within 0.01%.",
      "toleranceAmount": "0.01%",
      "material": true
    },
    {
      "ruleId": "ap-accruals-match",
      "accountPair": "2100/2200",
      "description": "AP and accruals accounts must net to zero.",
      "toleranceAmount": "0.00",
      "material": true
    }
  ],
  "submittedBy": "accountant-7"
}
```

### SignOffRequest (request body)

```json
{
  "approvedBy": "controller-3",
  "decision": "APPROVED",
  "note": "Variance in ap-accruals-match reviewed and approved — posting scheduled."
}
```

### Period (response body)

```json
{
  "periodId": "p-9ac...",
  "submission": {
    "periodId": "p-9ac...",
    "periodLabel": "Q2 FY2026",
    "rawBalance": "(raw CSV preserved for audit)",
    "rules": [
      { "ruleId": "cash-ar-offset", "accountPair": "1000/1200", "description": "...", "toleranceAmount": "0.01%", "material": true }
    ],
    "submittedBy": "accountant-7",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "sanitized": {
    "maskedBalance": "account_code,description,debit,credit\n1000,Cash,125000.00,0.00\n...",
    "maskedFieldTypes": ["margin-pct", "entity-code"]
  },
  "report": {
    "status": "HAS_VARIANCES",
    "summary": "Cash/AR offset is within tolerance; AP accruals balance is short by 4,200.00.",
    "findings": [
      {
        "ruleId": "cash-ar-offset",
        "accountPair": "1000/1200",
        "finding": "IN_BALANCE",
        "expectedBalance": 125000.00,
        "actualBalance": 124998.50,
        "variance": -1.50,
        "note": "Within 0.01% tolerance."
      }
    ],
    "proposedGlWrites": [
      "DR 2200 / CR 2100 4200.00 — Accruals shortfall Q2 FY2026"
    ],
    "reconciledAt": "2026-06-28T14:22:00Z"
  },
  "signOff": {
    "approvedBy": "controller-3",
    "decision": "APPROVED",
    "note": "Variance reviewed and approved.",
    "decidedAt": "2026-06-28T15:05:00Z"
  },
  "attestation": {
    "complete": true,
    "attestedBy": "controller-3",
    "attestedAt": "2026-06-28T15:05:01Z"
  },
  "status": "ATTESTED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T15:05:01Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: period-update
data: { "periodId": "p-9ac...", "status": "AWAITING_SIGNOFF", "report": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `RECONCILING`, `REPORT_RECORDED`, `AWAITING_SIGNOFF`, `ATTESTED`, `REJECTED`, `FAILED`). Clients reconcile by `periodId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` / `approvedBy` from the authenticated principal rather than the request body. The sign-off endpoint in particular should be restricted to roles with `finance-controller` or equivalent authority.
