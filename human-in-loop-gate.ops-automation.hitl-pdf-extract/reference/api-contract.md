# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/extraction-request` | `{ "documentUrl": "string" }` | `{ "documentId": "uuid" }` | ExtractionEndpoint → ExtractionWorkflow |
| POST | `/api/documents/{documentId}/approve` | `{ "reviewedBy": "string", "comment": "string" }` | `200` \| `404` | ExtractionEndpoint → DocumentEntity |
| POST | `/api/documents/{documentId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | ExtractionEndpoint → DocumentEntity |
| GET | `/api/documents` | — | `{ "documents": [Document, ...] }` | ExtractionEndpoint → DocumentsView |
| GET | `/api/documents/{documentId}` | — | `Document` \| `404` | ExtractionEndpoint → DocumentEntity |
| GET | `/api/documents/sse` | — | SSE stream of `Document` | ExtractionEndpoint → DocumentsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ExtractionEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ExtractionEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ExtractionEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Document` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "documentUrl": "string or null",
  "status": "EXTRACTED | PENDING_REVIEW | APPROVED | REJECTED | POSTED",
  "confidence": 0.0,
  "submittedAt": "ISO-8601 or null",
  "extractedAt": "ISO-8601 or null",
  "rawFields": { "fieldName": "rawValue", "..." : "..." },
  "redactedFields": { "fieldName": "redactedValue", "..." : "..." },
  "reviewedAt": "ISO-8601 or null",
  "reviewedBy": "string or null",
  "reviewerComment": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "postedAt": "ISO-8601 or null",
  "postTarget": "string or null"
}
```

Note: `rawFields` and `redactedFields` are `Map<String,String>` in Java. The UI displays only `redactedFields` to reviewers.

## SSE event format

`GET /api/documents/sse` streams the `DocumentsView` rows. Each message:

```
event: document
data: { ...Document... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
