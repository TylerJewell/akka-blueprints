# API contract — self-correcting-extraction

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "documentType": String, "rawText": String, "submittedBy"?: String }` | `202 { "jobId": String }` | `ExtractionEndpoint` → `DocumentQueue` |
| `GET` | `/api/jobs` | — | `200 [ ExtractionJob... ]` (optional `?status=EXTRACTING\|SCORING\|VERIFIED\|BUDGET_EXHAUSTED` — filtered client-side from `getAllJobs`) | `ExtractionEndpoint` ← `JobsView` |
| `GET` | `/api/jobs/{id}` | — | `200 ExtractionJob` / `404` | `ExtractionEndpoint` ← `JobsView` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` (one event per job change) | `ExtractionEndpoint` ← `JobsView` |
| `GET` | `/api/memory/{documentType}` | — | `200 { "documentType": String, "records": [ MemoryRecord... ] }` / `404` | `ExtractionEndpoint` ← `MemoryEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/jobs`

- Missing `submittedBy` → `"anonymous"`.
- `documentType` must be non-blank; otherwise `400`.
- `rawText` must be between 10 and 10 000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(documentType, submittedBy)`; the second submission returns the first `jobId` (200) instead of starting a new workflow.

## JSON shapes

### ExtractionJob

```json
{
  "jobId": "j-3a9f1…",
  "documentType": "invoice",
  "rawText": "Invoice #INV-2024-0042, dated 15 March 2024. Vendor: Acme Corp. Total: $1,250.00 USD.",
  "budgetCap": 4,
  "status": "VERIFIED",
  "attempts": [
    {
      "attemptNumber": 1,
      "fieldMap": {
        "fields": {
          "invoiceNumber": "INV-2024-0042",
          "invoiceDate": "03/15/2024",
          "vendorName": "Acme Corp",
          "totalAmount": "1,250.00",
          "currency": "USD"
        },
        "confidence": 0.55,
        "extractedAt": "2026-06-29T09:01:02Z"
      },
      "verdict": {
        "decision": "CORRECT",
        "notes": {
          "bullets": [
            "invoiceDate: got \"03/15/2024\"; expected ISO-8601 \"2024-03-15\".",
            "totalAmount: got \"1,250.00\"; expected numeric string without commas \"1250.00\"."
          ],
          "overallRationale": "Date and amount formats deviate from schema."
        },
        "confidence": 0.55,
        "scoredAt": "2026-06-29T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "fieldMap": {
        "fields": {
          "invoiceNumber": "INV-2024-0042",
          "invoiceDate": "2024-03-15",
          "vendorName": "Acme Corp",
          "totalAmount": "1250.00",
          "currency": "USD"
        },
        "confidence": 0.96,
        "extractedAt": "2026-06-29T09:01:21Z"
      },
      "verdict": {
        "decision": "PASS",
        "notes": {
          "bullets": [],
          "overallRationale": "All required fields present, formats correct, consistent with memory."
        },
        "confidence": 0.96,
        "scoredAt": "2026-06-29T09:01:33Z"
      }
    }
  ],
  "verifiedAttemptNumber": 2,
  "verifiedFieldMap": {
    "fields": {
      "invoiceNumber": "INV-2024-0042",
      "invoiceDate": "2024-03-15",
      "vendorName": "Acme Corp",
      "totalAmount": "1250.00",
      "currency": "USD"
    },
    "confidence": 0.96,
    "extractedAt": "2026-06-29T09:01:21Z"
  },
  "exhaustionReason": null,
  "createdAt": "2026-06-29T09:00:59Z",
  "finishedAt": "2026-06-29T09:01:34Z"
}
```

### MemoryRecord

```json
{
  "documentType": "invoice",
  "fieldName": "vendorName",
  "confirmedValue": "Acme Corp",
  "confirmedAt": "2026-06-29T09:01:34Z"
}
```

### SSE event format

```
event: job-update
data: { "jobId": "j-3a9f1…", "status": "SCORING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `jobId`. The full `ExtractionJob` JSON is included so a fresh client can render the row without a separate fetch.
