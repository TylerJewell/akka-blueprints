# API contract — sba-loan-processor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/loans` | `SubmitLoanRequest` | `201 { applicationId }` | `LoanEndpoint` → `LoanApplicationEntity` |
| `GET` | `/api/loans` | — | `200 [ LoanApplicationRecord... ]` (newest-first) | `LoanEndpoint` ← `LoanApplicationView` |
| `GET` | `/api/loans/{id}` | — | `200 LoanApplicationRecord` / `404` | `LoanEndpoint` ← `LoanApplicationView` |
| `GET` | `/api/loans/sse` | — | `text/event-stream` | `LoanEndpoint` ← `LoanApplicationView` |
| `POST` | `/api/loans/{id}/review` | `OfficerReviewRequest` | `200` / `409` if not `PENDING_REVIEW` | `LoanEndpoint` → `LoanPipelineWorkflow` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitLoanRequest (request body)

```json
{
  "applicantRef": "Sunrise Bakery / EIN 12-3456789",
  "loanAmount": 250000,
  "loanPurpose": "Equipment purchase and working capital"
}
```

### OfficerReviewRequest (request body for `POST /api/loans/{id}/review`)

```json
{
  "officerId": "officer-042",
  "outcome": "APPROVED",
  "notes": "DSCR is healthy; collateral is adequate. Conditions are standard."
}
```

`outcome` must be exactly `"APPROVED"` or `"DENIED"`. A `409 Conflict` is returned if the application is not in `PENDING_REVIEW` status.

### LoanApplicationRecord (response body)

```json
{
  "applicationId": "app-5af9c2...",
  "applicantRef": "Sunrise Bakery / EIN 12-3456789",
  "creditProfile": {
    "creditScore": {
      "tier": "GOOD",
      "score": 720,
      "fetchedAt": "2026-06-28T10:00:05Z"
    },
    "businessProfile": {
      "businessEin": "[SANITIZED]",
      "businessName": "Sunrise Bakery LLC",
      "industry": "Food & Beverage",
      "yearsInOperation": 7,
      "legalStructure": "LLC"
    },
    "profiledAt": "2026-06-28T10:00:08Z"
  },
  "analysis": {
    "cashFlow": {
      "annualRevenue": 480000,
      "operatingExpenses": 310000,
      "netOperatingIncome": 170000,
      "analyzedAt": "2026-06-28T10:00:22Z"
    },
    "collateral": {
      "collateralType": "commercial-equipment",
      "estimatedValue": 300000,
      "ltvRatio": 0.83
    },
    "keyRiskFactors": [
      "LTV ratio of 0.83 is above the 0.80 ceiling for TIER_2"
    ],
    "mitigants": [
      "7 years of continuous operation",
      "DSCR of 1.42 exceeds the 1.25 threshold"
    ],
    "analyzedAt": "2026-06-28T10:00:25Z"
  },
  "decision": {
    "recommendation": "APPROVE_WITH_CONDITIONS",
    "proposedAmount": 250000,
    "interestRateTier": "TIER_2",
    "rationale": "DSCR of 1.42 meets the threshold. LTV of 0.83 slightly exceeds the TIER_2 ceiling of 0.80; a personal guarantee is required to offset the collateral gap.",
    "conditions": [
      {
        "conditionId": "cond-personal-guarantee",
        "description": "Personal guarantee from primary owner required prior to disbursement.",
        "conditionType": "pre-disbursement"
      }
    ],
    "dscrResult": {
      "dscr": 1.42,
      "annualDebtService": 120000,
      "meetsThreshold": true
    },
    "riskTier": "TIER_2",
    "decidedAt": "2026-06-28T10:00:38Z"
  },
  "fairnessEval": {
    "score": 5,
    "flags": [],
    "rationale": "All four fair-lending checks passed.",
    "evaluatedAt": "2026-06-28T10:00:39Z"
  },
  "memo": null,
  "officerReview": null,
  "status": "PENDING_REVIEW",
  "submittedAt": "2026-06-28T10:00:00Z",
  "finishedAt": null,
  "guardrailRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the agent attempted a misordered tool call and the guardrail caught it.

Note: `businessProfile.businessEin` is serialised as `"[SANITIZED]"` in API responses — the raw EIN is stored only in the entity's `sensitiveData` map and is not surfaced through this endpoint.

### SSE event format

```
event: loan-update
data: { "applicationId": "app-5af9c2...", "status": "PENDING_REVIEW", "decision": { ... }, "fairnessEval": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `INTAKE_IN_PROGRESS`, `INTAKE_COMPLETE`, `UNDERWRITING`, `UNDERWRITING_COMPLETE`, `DECISION_IN_PROGRESS`, `DECISION_MADE`, `PENDING_REVIEW`, `OFFICER_APPROVED`, `OFFICER_DENIED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: loan-rejection
data: { "applicationId": "app-5af9c2...", "phase": "INTAKE", "tool": "computeDscr", "reason": "phase-violation: computeDscr requires status in {UNDERWRITING_COMPLETE, DECISION_IN_PROGRESS} with analysis present, saw INTAKE_IN_PROGRESS", "rejectedAt": "..." }
```

Clients reconcile by `applicationId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must:
- Wrap `POST /api/loans` with an authenticated `loanOfficer` or `submitter` role check.
- Gate `POST /api/loans/{id}/review` on an authenticated `loanOfficer` role; the `officerId` field must be populated from the authenticated principal, not the request body.
- Protect `GET /api/loans/{id}/sensitive` (the PII-bearing endpoint, if implemented) with an officer-or-admin role.
- Stamp a `submittedBy` field from the authenticated principal onto `SubmitLoanRequest` and extend `ApplicationSubmitted` to carry it.
