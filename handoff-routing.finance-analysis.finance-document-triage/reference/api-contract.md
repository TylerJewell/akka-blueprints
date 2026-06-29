# API contract — finance-document-triage

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/documents` | — | `200 [ Document... ]` (newest-first); supports optional `?docType=…&status=…` filtered client-side | `DocumentEndpoint` ← `DocumentView` |
| `GET` | `/api/documents/{id}` | — | `200 Document` / `404` | `DocumentEndpoint` ← `DocumentView` |
| `POST` | `/api/documents` | `{ "sourceId": String, "docFormat": String, "rawContent": String, "subject": String }` | `201 { "documentId": String }` | `DocumentEndpoint` → `DocumentQueue` |
| `POST` | `/api/documents/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Document` (status now `PROCESSED`) / `409` if not `BLOCKED` | `DocumentEndpoint` → `DocumentEntity` |
| `GET` | `/api/documents/sse` | — | `text/event-stream` | `DocumentEndpoint` ← `DocumentView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingDocument (request body for `POST /api/documents`, minus `documentId` and `receivedAt`)

```json
{
  "sourceId": "src-finance-drop-001",
  "docFormat": "pdf",
  "subject": "Invoice #INV-2024-0091 — professional services",
  "rawContent": "Vendor: Acme Corp. Account: ACCT-882901. Tax ID: GB123456789. Amount: $4,200. Due: 30 days."
}
```

### Document

```json
{
  "documentId": "doc-3cf91a22",
  "incoming": {
    "documentId": "doc-3cf91a22",
    "sourceId": "src-finance-drop-001",
    "docFormat": "pdf",
    "subject": "Invoice #INV-2024-0091 — professional services",
    "rawContent": "...raw with PII and sector data...",
    "receivedAt": "2026-06-28T08:10:00Z"
  },
  "sanitized": {
    "redactedContent": "Vendor: [REDACTED-NAME]. Account: [REDACTED-ACCOUNT]. Tax ID: [REDACTED-TAX-ID]. Amount: $4,200. Due: 30 days.",
    "redactedSubject": "Invoice #INV-2024-0091 — professional services",
    "piiCategoriesFound": ["name", "account-number", "tax-id"],
    "sectorFieldsRedacted": []
  },
  "classification": {
    "docType": "INVOICE",
    "confidence": "high",
    "reason": "Numbered invoice with explicit payment term and amount."
  },
  "result": {
    "resultSummary": "Invoice received and approved for payment processing.",
    "resultDetail": "The invoice covers professional services billed at $4,200 with a 30-day payment term...",
    "action": "APPROVED",
    "processorTag": "invoice",
    "referenceId": null,
    "processedAt": "2026-06-28T08:10:22Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "classificationScore": {
    "score": 5,
    "rationale": "Type right; reason names the structural invoice marker and payment term directly.",
    "scoredAt": "2026-06-28T08:10:14Z"
  },
  "escalationReason": null,
  "status": "PROCESSED",
  "createdAt": "2026-06-28T08:10:00Z",
  "finishedAt": "2026-06-28T08:10:23Z"
}
```

A blocked document has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated document has `escalationReason` populated, `status = "ESCALATED"`.

### ClassificationDecision (returned by `ClassifierAgent`, embedded in `Document.classification`)

```json
{ "docType": "INVOICE" | "LOAN_APPLICATION" | "COMPLIANCE_REVIEW",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### ProcessingResult (returned by any processor, embedded in `Document.result`)

```json
{ "resultSummary": "one-sentence summary ≤ 100 chars",
  "resultDetail": "2–4 short paragraphs",
  "action": "APPROVED" | "FLAGGED_FOR_REVIEW" | "INFORMATION_REQUESTED" | "REJECTED" | "FORWARDED_TO_COMPLIANCE" | "ESCALATED",
  "processorTag": "invoice" | "loan" | "compliance",
  "referenceId": "optional-string-or-null",
  "processedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `OutputGuardrail`, embedded in `Document.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["invented-approval-amount", "fabricated-credit-conclusion", …],
  "rubricVersion": "v1" }
```

### ClassificationScore (returned by `ClassificationJudge`, embedded in `Document.classificationScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: document-update
data: { "documentId": "doc-…", "status": "CLASSIFIED", … full Document JSON … }
```

One event per state transition on the `DocumentView`. Clients reconcile by `documentId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/documents`).
- `404` — unknown `documentId`.
- `409` — `unblock` requested on a document not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
