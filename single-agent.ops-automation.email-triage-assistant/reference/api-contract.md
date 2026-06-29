# API contract — email-triage-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/emails` | `SubmitEmailRequest` | `201 { emailId }` | `TriageEndpoint` → `EmailEntity` |
| `GET` | `/api/emails` | — | `200 [ Email... ]` (newest-first) | `TriageEndpoint` ← `TriageView` |
| `GET` | `/api/emails/{id}` | — | `200 Email` / `404` | `TriageEndpoint` ← `TriageView` |
| `GET` | `/api/emails/sse` | — | `text/event-stream` | `TriageEndpoint` ← `TriageView` |
| `POST` | `/api/emails/{id}/confirm` | `ConfirmRequest` | `204` | `TriageEndpoint` → `ConfirmationEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitEmailRequest (request body)

```json
{
  "fromAddress": "sender@example.com",
  "subject": "Unexpected charge on my account — ref #INV-4821",
  "rawBody": "Hi, I noticed a charge of $149 on my statement dated June 15. I did not authorise this. My account number is ACC-8820. Please refund. — Jane Doe, jane.doe@personal.net, +1 650 555 0199",
  "attachments": [
    {
      "filename": "billing-statement.pdf",
      "textContent": "Invoice #INV-4821 dated 2026-06-15. Account holder: Jane Doe. Charge: $149.00 for Premium Plan renewal. Card ending 4242."
    }
  ],
  "submittedBy": "ops-agent-7"
}
```

### Email (response body)

```json
{
  "emailId": "e-7af...",
  "submission": {
    "emailId": "e-7af...",
    "fromAddress": "sender@example.com",
    "subject": "Unexpected charge on my account — ref #INV-4821",
    "rawBody": "(raw body preserved for audit)",
    "attachments": [
      { "filename": "billing-statement.pdf", "textContent": "(raw attachment text preserved for audit)" }
    ],
    "submittedBy": "ops-agent-7",
    "submittedAt": "2026-06-28T09:14:00Z"
  },
  "sanitized": {
    "redactedBody": "Hi, I noticed a charge of $149 on my statement dated June 15. I did not authorise this. My account number is [REDACTED-ACCOUNT]. Please refund. — [REDACTED-NAME], [REDACTED-EMAIL], [REDACTED-PHONE]",
    "redactedAttachmentTexts": [
      "Invoice #INV-4821 dated 2026-06-15. Account holder: [REDACTED-NAME]. Charge: $149.00 for Premium Plan renewal. Card ending [REDACTED-PCN]."
    ],
    "piiCategoriesFound": ["email", "phone", "person-name", "account-id", "payment-card-number"]
  },
  "triage": {
    "urgency": "HIGH",
    "category": "BILLING",
    "classificationRationale": "Sender reports an unauthorised charge and requests a refund; financial dispute with explicit account reference elevates urgency to HIGH.",
    "draftReply": "Dear Customer,\n\nThank you for contacting us about the charge on your account (ref #INV-4821). We have received your dispute and our billing team will review the transaction within 2 business days. You will receive a follow-up email with our findings.\n\nBest regards,\nThe Support Team",
    "classifiedAt": "2026-06-28T09:14:22Z"
  },
  "confirmation": {
    "emailId": "e-7af...",
    "decision": "APPROVED",
    "decidedBy": "ops-agent-7",
    "decidedAt": "2026-06-28T09:15:05Z"
  },
  "status": "SENT",
  "createdAt": "2026-06-28T09:14:00Z",
  "finishedAt": "2026-06-28T09:15:08Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### ConfirmRequest (request body)

```json
{
  "decision": "APPROVED",
  "decidedBy": "ops-agent-7"
}
```

`decision` must be one of `"APPROVED"` or `"REJECTED"`. Returns `204 No Content` on success.

### SSE event format

```
event: email-update
data: { "emailId": "e-7af...", "status": "DRAFT_READY", "triage": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `REVIEWING`, `DRAFT_READY`, `SENDING`, `SENT`, `REJECTED`, `FAILED`). Clients reconcile by `emailId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware, set `submittedBy` from the authenticated principal rather than the request body, and validate that only the submitting user (or an authorized supervisor) can call `POST /api/emails/{id}/confirm`.
