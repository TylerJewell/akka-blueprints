# API contract — invoice-processing

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/invoices` | `SubmitInvoiceRequest` | `201 { invoiceId }` | `InvoiceEndpoint` → `InvoiceEntity` |
| `GET` | `/api/invoices` | — | `200 [ InvoiceRecord... ]` (newest-first) | `InvoiceEndpoint` ← `InvoiceView` |
| `GET` | `/api/invoices/{id}` | — | `200 InvoiceRecord` / `404` | `InvoiceEndpoint` ← `InvoiceView` |
| `GET` | `/api/invoices/sse` | — | `text/event-stream` | `InvoiceEndpoint` ← `InvoiceView` |
| `POST` | `/api/invoices/{id}/approve` | `ApprovalDecisionRequest` | `200 { invoiceId, status }` / `409` | `InvoiceEndpoint` → `InvoiceEntity` |
| `POST` | `/api/invoices/{id}/deny` | `ApprovalDecisionRequest` | `200 { invoiceId, status }` / `409` | `InvoiceEndpoint` → `InvoiceEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitInvoiceRequest (request body)

```json
{
  "rawText": "INVOICE\nVendor: Acme Corp\nbilling@acme.example.com\n+1-555-0100\nDate: 2026-06-15\nINV-2026-0421\nCloud storage Q2: 1 x 2400.00\nSupport renewal: 2 x 400.00\nTotal: 3200.00 USD"
}
```

### ApprovalDecisionRequest (request body)

```json
{
  "reviewerNote": "PO-2026-0088 verified; amount approved."
}
```

A `409 Conflict` is returned if the invoice is not in `APPROVAL_REQUESTED` state when approve or deny is called.

### InvoiceRecord (response body)

```json
{
  "invoiceId": "inv-7bc3d1...",
  "rawText": "INVOICE\nVendor: Acme Corp\n...",
  "extracted": {
    "header": {
      "invoiceNumber": "INV-2026-0421",
      "vendorName": "Acme Corp",
      "vendorEmail": "[REDACTED]",
      "vendorPhone": "[REDACTED]",
      "invoiceDate": "2026-06-15",
      "currency": "USD",
      "totalAmount": 3200.00
    },
    "lineItems": [
      { "lineId": "li-1", "description": "Cloud storage Q2", "quantity": 1, "unitPrice": 2400.00, "lineTotal": 2400.00 },
      { "lineId": "li-2", "description": "Support renewal", "quantity": 2, "unitPrice": 400.00, "lineTotal": 800.00 }
    ],
    "extractedAt": "2026-06-28T10:00:02Z"
  },
  "sanitization": {
    "redactedFields": ["vendorEmail", "vendorPhone"],
    "sanitizedAt": "2026-06-28T10:00:02Z"
  },
  "validated": {
    "source": { "...": "same as extracted above" },
    "lineItems": [
      {
        "lineItem": { "lineId": "li-1", "description": "Cloud storage Q2", "quantity": 1, "unitPrice": 2400.00, "lineTotal": 2400.00 },
        "glAccount": { "accountCode": "6200", "accountName": "Cloud Infrastructure", "accountType": "EXPENSE" },
        "balanceOk": true
      },
      {
        "lineItem": { "lineId": "li-2", "description": "Support renewal", "quantity": 2, "unitPrice": 400.00, "lineTotal": 800.00 },
        "glAccount": { "accountCode": "6400", "accountName": "Software Support", "accountType": "EXPENSE" },
        "balanceOk": true
      }
    ],
    "balance": { "balanced": true, "debitTotal": 3200.00, "creditTotal": 3200.00 },
    "totalAmount": 3200.00,
    "validatedAt": "2026-06-28T10:00:10Z"
  },
  "posted": {
    "validated": { "...": "same as validated above" },
    "journalEntry": {
      "entryId": "je-a1b2c3",
      "invoiceRef": "INV-2026-0421",
      "lines": [
        { "accountCode": "6200", "accountName": "Cloud Infrastructure", "debit": 2400.00, "credit": 0 },
        { "accountCode": "6400", "accountName": "Software Support", "debit": 800.00, "credit": 0 },
        { "accountCode": "2000", "accountName": "Accounts Payable", "debit": 0, "credit": 3200.00 }
      ],
      "entryDate": "2026-06-28"
    },
    "confirmation": {
      "confirmationRef": "conf-5e2a1f3b",
      "postedAt": "2026-06-28T10:00:18Z"
    },
    "postedAt": "2026-06-28T10:00:18Z"
  },
  "eval": {
    "score": 5,
    "rationale": "GL coverage, journal balance, line parity, and posting confirmation all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:18Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:18Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). Note that `vendorEmail` and `vendorPhone` are always `"[REDACTED]"` in any API response — the sanitizer runs before the first entity write, so no raw PII ever appears in the API.

### SSE event format

```
event: invoice-update
data: { "invoiceId": "inv-7bc3d1...", "status": "VALIDATED", "validated": { ... }, ... }
```

One event per state transition (`CREATED`, `EXTRACTING`, `EXTRACTED`, `VALIDATING`, `VALIDATED`, `APPROVAL_REQUESTED`, `APPROVED`, `POSTING`, `POSTED`, `EVALUATED`, `DENIED`, `FAILED`) and one per `PhaseGuardrailRejected` audit event:

```
event: invoice-rejection
data: { "invoiceId": "inv-7bc3d1...", "phase": "EXTRACT", "tool": "buildJournalEntry", "reason": "phase-violation: buildJournalEntry requires status in {APPROVED, POSTING}, saw EXTRACTING", "rejectedAt": "..." }
```

Clients reconcile by `invoiceId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitInvoiceRequest` record and the `InvoiceCreated` event to carry it. The approve/deny endpoints must additionally verify that the calling principal holds a `finance-reviewer` role; add role-check middleware before calling `InvoiceEntity.approve()` / `InvoiceEntity.deny()`.
