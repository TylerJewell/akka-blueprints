# API contract — schema-extractor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `SubmitJobRequest` | `201 { jobId }` | `ExtractionEndpoint` → `ExtractionJobEntity` |
| `GET` | `/api/jobs` | — | `200 [ ExtractionJob... ]` (newest-first) | `ExtractionEndpoint` ← `ExtractionView` |
| `GET` | `/api/jobs/{id}` | — | `200 ExtractionJob` / `404` | `ExtractionEndpoint` ← `ExtractionView` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` | `ExtractionEndpoint` ← `ExtractionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitJobRequest (request body)

```json
{
  "documentTitle": "Meridian Supplies — Invoice INV-2026-00441",
  "rawDocument": "INVOICE\n\nFrom: Meridian Supplies Ltd., 42 Commerce Park, Bristol BS1 4QZ\nTo: Acme Corp, jane.doe@acme.example ...",
  "targetSchema": {
    "schemaId": "invoice-schema",
    "schemaName": "Vendor Invoice",
    "fields": [
      { "fieldName": "invoiceNumber", "fieldType": "string", "required": true,  "maxLength": 50  },
      { "fieldName": "vendorName",    "fieldType": "string", "required": true,  "maxLength": 100 },
      { "fieldName": "vendorTaxId",   "fieldType": "string", "required": false, "maxLength": 30  },
      { "fieldName": "issueDate",     "fieldType": "date",   "required": true,  "maxLength": 0   },
      { "fieldName": "dueDate",       "fieldType": "date",   "required": true,  "maxLength": 0   },
      { "fieldName": "currency",      "fieldType": "string", "required": true,  "maxLength": 3   },
      { "fieldName": "subtotal",      "fieldType": "number", "required": true,  "maxLength": 0   },
      { "fieldName": "taxAmount",     "fieldType": "number", "required": false, "maxLength": 0   },
      { "fieldName": "totalAmount",   "fieldType": "number", "required": true,  "maxLength": 0   },
      { "fieldName": "paymentTerms",  "fieldType": "string", "required": true,  "maxLength": 200 }
    ]
  },
  "submittedBy": "ops-pipeline-01"
}
```

### ExtractionJob (response body)

```json
{
  "jobId": "j-7f3a...",
  "request": {
    "jobId": "j-7f3a...",
    "documentTitle": "Meridian Supplies — Invoice INV-2026-00441",
    "rawDocument": "(raw text preserved for audit)",
    "targetSchema": { "schemaId": "invoice-schema", "schemaName": "Vendor Invoice", "fields": [ "..." ] },
    "submittedBy": "ops-pipeline-01",
    "submittedAt": "2026-06-28T14:20:00Z"
  },
  "sanitized": {
    "redactedDocument": "INVOICE\n\nFrom: Meridian Supplies Ltd., 42 Commerce Park, Bristol BS1 4QZ\nTo: Acme Corp, [REDACTED-EMAIL] ...",
    "piiCategoriesFound": ["email", "address"]
  },
  "record": {
    "schemaId": "invoice-schema",
    "fields": [
      { "fieldName": "invoiceNumber", "fieldType": "string", "value": "INV-2026-00441",      "confidence": "HIGH"   },
      { "fieldName": "vendorName",    "fieldType": "string", "value": "Meridian Supplies Ltd.", "confidence": "HIGH" },
      { "fieldName": "vendorTaxId",   "fieldType": "string", "value": "GB123456789",          "confidence": "HIGH"   },
      { "fieldName": "issueDate",     "fieldType": "date",   "value": "2026-06-15",            "confidence": "HIGH"   },
      { "fieldName": "dueDate",       "fieldType": "date",   "value": "2026-07-15",            "confidence": "HIGH"   },
      { "fieldName": "currency",      "fieldType": "string", "value": "GBP",                  "confidence": "HIGH"   },
      { "fieldName": "subtotal",      "fieldType": "number", "value": "2840.00",              "confidence": "HIGH"   },
      { "fieldName": "taxAmount",     "fieldType": "number", "value": "340.80",               "confidence": "MEDIUM" },
      { "fieldName": "totalAmount",   "fieldType": "number", "value": "3180.80",              "confidence": "HIGH"   },
      { "fieldName": "paymentTerms",  "fieldType": "string", "value": "Net 30 days",          "confidence": "HIGH"   }
    ],
    "schemaCoveragePercent": 100,
    "extractedAt": "2026-06-28T14:20:22Z"
  },
  "status": "RECORD_EXTRACTED",
  "createdAt": "2026-06-28T14:20:00Z",
  "finishedAt": "2026-06-28T14:20:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: job-update
data: { "jobId": "j-7f3a...", "status": "RECORD_EXTRACTED", "record": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `EXTRACTING`, `RECORD_EXTRACTED`, `FAILED`). Clients reconcile by `jobId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
