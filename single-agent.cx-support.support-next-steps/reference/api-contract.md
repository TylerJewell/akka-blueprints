# API contract — support-next-steps

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tickets` | `SubmitTicketRequest` | `201 { ticketId }` | `TicketEndpoint` → `TicketEntity` |
| `GET` | `/api/tickets` | — | `200 [ Ticket... ]` (newest-first) | `TicketEndpoint` ← `TicketView` |
| `GET` | `/api/tickets/{id}` | — | `200 Ticket` / `404` | `TicketEndpoint` ← `TicketView` |
| `GET` | `/api/tickets/sse` | — | `text/event-stream` | `TicketEndpoint` ← `TicketView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTicketRequest (request body)

```json
{
  "subject": "Unexpected charge on invoice #INV-2026-0441",
  "ticketBody": "Hi support, I was charged $149.00 on June 15 but my plan is $99/month. My account email is jane.doe@acme.com and my customer ID is CUST-88421. Please investigate.",
  "productArea": "Billing",
  "submittedBy": "support-agent-7"
}
```

### Ticket (response body)

```json
{
  "ticketId": "t-4ac...",
  "request": {
    "ticketId": "t-4ac...",
    "subject": "Unexpected charge on invoice #INV-2026-0441",
    "ticketBody": "(raw text preserved for audit)",
    "productArea": "Billing",
    "submittedBy": "support-agent-7",
    "submittedAt": "2026-06-28T10:15:00Z"
  },
  "sanitized": {
    "redactedBody": "Hi support, I was charged $149.00 on June 15 but my plan is $99/month. My account email is [REDACTED-EMAIL] and my customer ID is [REDACTED-ACCOUNT]. Please investigate.",
    "piiCategoriesFound": ["email", "account-number"]
  },
  "recommendation": {
    "overallConfidence": "HIGH",
    "rationale": "The described charge discrepancy matches BIL-2024-09 closely; the two-step verification and refund path resolved that case within one business day.",
    "steps": [
      {
        "stepNumber": 1,
        "description": "Verify the charge date and amount against the billing ledger for the customer account",
        "actionType": "VERIFY",
        "confidence": "HIGH",
        "resolutionRef": "BIL-2024-09"
      },
      {
        "stepNumber": 2,
        "description": "Initiate a refund request if the charge is confirmed as a duplicate",
        "actionType": "CONFIGURE",
        "confidence": "HIGH",
        "resolutionRef": "BIL-2024-09"
      }
    ],
    "decidedAt": "2026-06-28T10:15:22Z"
  },
  "eval": {
    "score": 5,
    "rationale": "All steps cite resolution references, descriptions are actionable, and confidence levels vary across steps.",
    "evaluatedAt": "2026-06-28T10:15:23Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:15:00Z",
  "finishedAt": "2026-06-28T10:15:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: ticket-update
data: { "ticketId": "t-4ac...", "status": "RECOMMENDATION_RECORDED", "recommendation": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `ADVISING`, `RECOMMENDATION_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `ticketId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
