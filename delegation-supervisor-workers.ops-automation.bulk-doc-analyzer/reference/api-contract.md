# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api/documents`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/documents/batches` | `{ "rawDocuments": ["string", ...], "submittedBy": "string" }` | `{ "batchId": "uuid" }` | `DocumentEndpoint` → `BatchQueue` |
| GET | `/api/documents/batches` | — | `{ "batches": [BatchRow, ...] }` | `DocumentEndpoint` → `DocumentView` |
| GET | `/api/documents/batches/{batchId}` | — | `BatchRow` with embedded documents or 404 | `DocumentEndpoint` → `DocumentView` |
| GET | `/api/documents` | — | `{ "documents": [DocumentRow, ...] }` | `DocumentEndpoint` → `DocumentView` |
| GET | `/api/documents?status=PROCESSED` | — | filtered list (client-side filter) | `DocumentEndpoint` |
| GET | `/api/documents/{id}` | — | `DocumentRow` or 404 | `DocumentEndpoint` |
| GET | `/api/documents/sse` | — | `text/event-stream` of `DocumentRow` | `DocumentEndpoint` → `DocumentView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `DocumentEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `DocumentEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `DocumentEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/documents/batches` request:

```json
{
  "rawDocuments": [
    "INVOICE REF-2025-0042\nDate: 2025-03-15\nBilled to: Acme Corp\n...",
    "POLICY NOTICE\nIssued by: Regulatory Authority\nDate: 2025-03-10\n..."
  ],
  "submittedBy": "ops-team"
}
```

`DocumentRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "documentId": "uuid",
  "batchId": "uuid",
  "status": "QUEUED | PROCESSING | PROCESSED | DEGRADED",
  "fields": {
    "documentDate": "2025-03-15",
    "author": "Acme Corp",
    "referenceNumber": "REF-2025-0042",
    "bodyText": "...",
    "extractedAt": "ISO-8601"
  },
  "classification": {
    "riskCategory": "MEDIUM",
    "sensitivityLevel": "INTERNAL",
    "flaggedTopics": ["commercial contract", "payment terms"],
    "classifiedAt": "ISO-8601"
  },
  "processed": {
    "sanitizedText": "INVOICE ...\nBilled to: [REDACTED:person-name]\n...",
    "piiRedactions": [
      { "fieldName": "bodyText", "piiType": "person-name", "replacement": "[REDACTED:person-name]" }
    ],
    "processedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "qualityScore": "1-5 or null",
  "qualityRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

`BatchRow`:

```json
{
  "batchId": "uuid",
  "submittedBy": "ops-team",
  "status": "PENDING | IN_PROGRESS | COMPLETE | PARTIAL",
  "totalDocuments": 3,
  "processedCount": 2,
  "degradedCount": 1,
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/documents/sse` emits one event per document change:

```
event: document
data: { ...DocumentRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `documentId`. No polling.
