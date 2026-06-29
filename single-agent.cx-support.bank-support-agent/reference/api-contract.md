# API contract — bank-support-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/enquiries` | `SubmitEnquiryRequest` | `201 { enquiryId }` | `EnquiryEndpoint` → `EnquiryEntity` + `EnquiryWorkflow` |
| `GET` | `/api/enquiries` | — | `200 [ EnquiryRow... ]` (newest-first) | `EnquiryEndpoint` ← `EnquiryView` |
| `GET` | `/api/enquiries/{id}` | — | `200 Enquiry` / `404` | `EnquiryEndpoint` ← `EnquiryEntity` |
| `GET` | `/api/enquiries/sse` | — | `text/event-stream` | `EnquiryEndpoint` ← `EnquiryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitEnquiryRequest (request body)

```json
{
  "customerId": "cust-0041",
  "enquiryText": "I think my card has been lost. I haven't seen it since yesterday and I'm worried someone might use it.",
  "category": "LOST_CARD",
  "submittedBy": "support-agent-7"
}
```

`category` is one of: `BALANCE_QUERY`, `DISPUTED_TRANSACTION`, `LOST_CARD`, `GENERAL`.

### Enquiry (response body — full entity, returned by GET /api/enquiries/{id})

```json
{
  "enquiryId": "enq-7a3f...",
  "enquiry": {
    "enquiryId": "enq-7a3f...",
    "customerId": "cust-0041",
    "enquiryText": "I think my card has been lost ...",
    "category": "LOST_CARD",
    "submittedBy": "support-agent-7",
    "submittedAt": "2026-06-28T14:22:00Z"
  },
  "account": {
    "customerId": "cust-0041",
    "maskedAccountNumber": "****4821",
    "accountType": "CHECKING",
    "availableBalance": 1240.00,
    "currencyCode": "GBP",
    "cardActive": true
  },
  "response": {
    "answer": "We've noted your report that the card on account ****4821 has been lost. Your current available balance is £1,240.00. We strongly recommend deactivating the card immediately — a replacement will be dispatched within 3–5 working days.",
    "riskScore": 8,
    "blockCard": true,
    "tone": "URGENT",
    "decidedAt": "2026-06-28T14:22:18Z"
  },
  "sanitizedLog": {
    "redactedAnswer": "We've noted your report that the card on account [REDACTED-ACCOUNT] has been lost. Your current available balance is [REDACTED-BALANCE]. We strongly recommend deactivating the card immediately — a replacement will be dispatched within 3–5 working days.",
    "piiCategoriesRedacted": ["account-number", "balance"]
  },
  "status": "LOGGED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### EnquiryRow (response body — view row, returned by GET /api/enquiries and SSE)

```json
{
  "enquiryId": "enq-7a3f...",
  "customerId": "cust-0041",
  "category": "LOST_CARD",
  "maskedAccountNumber": "****4821",
  "riskScore": 8,
  "blockCard": true,
  "tone": "URGENT",
  "redactedAnswer": "We've noted your report that the card on account [REDACTED-ACCOUNT] has been lost ...",
  "piiCategoriesRedacted": ["account-number", "balance"],
  "status": "LOGGED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:19Z"
}
```

The view row never includes `response.answer` (raw). Callers that need the raw response for compliance review fetch `GET /api/enquiries/{id}`.

### SSE event format

```
event: enquiry-update
data: { "enquiryId": "enq-7a3f...", "status": "LOGGED", "riskScore": 8, "blockCard": true, ... }
```

One event per state transition (`SUBMITTED`, `ACCOUNT_LOADED`, `RESPONDING`, `RESPONSE_RECORDED`, `LOGGED`, `FAILED`). Events carry the full `EnquiryRow` at the moment of transition so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal. The `GET /api/enquiries/{id}` endpoint returns the raw `response.answer` and should be gated behind a compliance role in production.
