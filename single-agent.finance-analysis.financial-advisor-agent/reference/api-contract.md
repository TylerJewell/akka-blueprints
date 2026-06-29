# API contract — financial-advisor-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/advisories` | `SubmitAdvisoryRequest` | `201 { advisoryId }` | `AdvisoryEndpoint` → `AdvisoryEntity` |
| `GET` | `/api/advisories` | — | `200 [ Advisory... ]` (newest-first) | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/advisories/{id}` | — | `200 Advisory` / `404` | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/advisories/sse` | — | `text/event-stream` | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAdvisoryRequest (request body)

```json
{
  "profile": {
    "clientId": "client-7291",
    "accountType": "IRA",
    "riskTolerance": "CONSERVATIVE",
    "holdings": [
      {
        "ticker": "VTI",
        "assetClass": "ETF",
        "currentWeight": 0.65,
        "marketValue": 81250.00
      },
      {
        "ticker": "AGG",
        "assetClass": "fixed-income",
        "currentWeight": 0.20,
        "marketValue": 25000.00
      },
      {
        "ticker": "GOOGL",
        "assetClass": "equities",
        "currentWeight": 0.15,
        "marketValue": 18750.00
      }
    ],
    "submittedBy": "advisor-42"
  },
  "question": "Should I reduce my equity exposure given my upcoming retirement in 5 years?"
}
```

### Advisory (response body)

```json
{
  "advisoryId": "a-3cf...",
  "request": {
    "advisoryId": "a-3cf...",
    "profile": {
      "clientId": "[REDACTED-ACCOUNT]",
      "accountType": "IRA",
      "riskTolerance": "CONSERVATIVE",
      "holdings": [
        { "ticker": "VTI", "assetClass": "ETF", "currentWeight": 0.65, "marketValue": 81250.00 },
        { "ticker": "AGG", "assetClass": "fixed-income", "currentWeight": 0.20, "marketValue": 25000.00 },
        { "ticker": "GOOGL", "assetClass": "equities", "currentWeight": 0.15, "marketValue": 18750.00 }
      ],
      "submittedBy": "advisor-42"
    },
    "question": "Should I reduce my equity exposure given my upcoming retirement in 5 years?",
    "requestedAt": "2026-06-28T12:34:00Z"
  },
  "sanitized": {
    "redactedProfileJson": "{ \"clientId\": \"[REDACTED-ACCOUNT]\", \"accountType\": \"IRA\", ... }",
    "identifierCategoriesFound": ["account-id"]
  },
  "response": {
    "riskRating": "CONSERVATIVE",
    "recommendation": "Reduce equity exposure by trimming VTI and reallocating to AGG ahead of retirement. The current 65% equity weight is elevated for a conservative IRA with a 5-year horizon; shifting 15 points toward investment-grade bonds reduces sequence-of-returns risk.",
    "holdingAdvice": [
      {
        "ticker": "VTI",
        "assetClass": "ETF",
        "currentWeight": 0.65,
        "suggestedWeight": 0.50,
        "rationale": "Trim to 50% to reduce equity beta as retirement approaches."
      },
      {
        "ticker": "AGG",
        "assetClass": "fixed-income",
        "currentWeight": 0.20,
        "suggestedWeight": 0.35,
        "rationale": "Increase to 35% to provide stable income and reduce volatility."
      },
      {
        "ticker": "GOOGL",
        "assetClass": "equities",
        "currentWeight": 0.15,
        "suggestedWeight": 0.15,
        "rationale": "Hold; single-stock allocation is small enough to retain without adding concentration risk."
      }
    ],
    "advisedAt": "2026-06-28T12:34:18Z"
  },
  "status": "AUDITED",
  "createdAt": "2026-06-28T12:34:00Z",
  "finishedAt": "2026-06-28T12:34:20Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: advisory-update
data: { "advisoryId": "a-3cf...", "status": "RESPONSE_RECORDED", "response": { ... }, ... }
```

One event per state transition (`REQUESTED`, `SANITIZED`, `ADVISING`, `RESPONSE_RECORDED`, `AUDITED`, `FAILED`). Clients reconcile by `advisoryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
