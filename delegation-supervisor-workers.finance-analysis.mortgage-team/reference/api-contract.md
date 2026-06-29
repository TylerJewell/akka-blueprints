# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/applications` | `ApplicationRequest` (see below) | `{ "applicationId": "uuid" }` | `MortgageEndpoint` → `ApplicationQueue` |
| GET | `/api/applications` | — | `{ "applications": [ApplicationRow, ...] }` | `MortgageEndpoint` → `ApplicationView` |
| GET | `/api/applications?status=PENDING_SIGNOFF` | — | filtered list (client-side filter) | `MortgageEndpoint` |
| GET | `/api/applications/{id}` | — | `ApplicationRow` or 404 | `MortgageEndpoint` |
| POST | `/api/applications/{id}/decision` | `{ "approved": boolean, "decidedBy": "string", "notes": "string" }` | `{ "applicationId": "uuid", "status": "APPROVED\|DECLINED" }` | `MortgageEndpoint` → `ApplicationEntity` |
| GET | `/api/applications/sse` | — | `text/event-stream` of `ApplicationRow` | `MortgageEndpoint` → `ApplicationView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `MortgageEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `MortgageEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `MortgageEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/applications` request body (`ApplicationRequest`):

```json
{
  "applicantRef": "APP-2024-00123",
  "loanProduct": "30-year-fixed",
  "requestedAmountGBP": 320000,
  "propertyValueGBP": 400000,
  "employmentStatus": "FULL_TIME",
  "annualIncomeGBP": 75000,
  "creditScore": 720,
  "documentIds": ["P60-2024", "BS-3MONTH-LLOYDS", "PHOTO-ID-PASSPORT", "POA-UTILITY-2024", "VALUATION-2024"]
}
```

`POST /api/applications/{id}/decision` request body:

```json
{
  "approved": true,
  "decidedBy": "underwriter.jane.smith@lender.example",
  "notes": "Standard residential; conditions met. Proceed to formal offer."
}
```

`ApplicationRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "applicationId": "uuid",
  "applicantRef": "APP-2024-00123",
  "loanProduct": "30-year-fixed",
  "requestedAmountGBP": 320000,
  "status": "RECEIVED | UNDER_REVIEW | PENDING_SIGNOFF | APPROVED | DECLINED | COMPLIANCE_HOLD | REVIEW_REQUIRED",
  "packet": {
    "eligibilityVerdict": {
      "eligible": true,
      "ltv": 0.80,
      "dti": 0.31,
      "verdict": "Application meets standard LTV and DTI thresholds.",
      "assessedAt": "ISO-8601"
    },
    "documentationVerdict": {
      "complete": true,
      "missingDocuments": [],
      "authenticityFlags": [],
      "reviewedAt": "ISO-8601"
    },
    "underwritingAssessment": {
      "riskBand": "MEDIUM",
      "proposedTerms": {
        "annualRatePercent": 5.25,
        "amortisationMonths": 300,
        "conditions": "Standard residential; no additional conditions."
      },
      "rationale": "LTV of 0.80 and credit score of 720 place this application in MEDIUM band.",
      "assessedAt": "ISO-8601"
    },
    "complianceStatus": "PASS",
    "sanitisedSummary": "Application presents a standard residential case ...",
    "packetAt": "ISO-8601"
  },
  "complianceHoldReason": "string or null",
  "decisionBy": "string or null",
  "decisionNotes": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "receivedAt": "ISO-8601",
  "decidedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/applications/sse` emits one event per application change:

```
event: application
data: { ...ApplicationRow... }

```

The App UI subscribes via `EventSource` and upserts each row into its live list by `applicationId`. No polling. The HITL decision panel appears on rows whose `status` is `PENDING_SIGNOFF`, enabling the underwriter to POST a decision without leaving the interface.
