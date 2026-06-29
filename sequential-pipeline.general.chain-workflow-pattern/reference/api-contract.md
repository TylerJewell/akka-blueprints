# API contract — chain-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/documents` | `SubmitDocumentRequest` | `201 { documentId }` | `DocumentEndpoint` → `DocumentEntity` |
| `GET` | `/api/documents` | — | `200 [ DocumentRecord... ]` (newest-first) | `DocumentEndpoint` ← `DocumentView` |
| `GET` | `/api/documents/{id}` | — | `200 DocumentRecord` / `404` | `DocumentEndpoint` ← `DocumentView` |
| `GET` | `/api/documents/sse` | — | `text/event-stream` | `DocumentEndpoint` ← `DocumentView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitDocumentRequest (request body)

```json
{
  "rawInput": "Product release notes draft: The new export feature supports CSV and JSON formats. Batch export is limited to 10,000 rows per request."
}
```

### DocumentRecord (response body)

```json
{
  "documentId": "d-5af9c2...",
  "rawInput": "Product release notes draft: ...",
  "extracted": {
    "facts": [
      {
        "factId": "f-3a1b22c0",
        "text": "The new export feature supports CSV and JSON formats.",
        "category": "feature",
        "sourceSpan": "0-52"
      }
    ],
    "extractedAt": "2026-06-28T10:00:00Z"
  },
  "refined": {
    "refinedFacts": [
      {
        "refinedId": "r-3a1b22c0",
        "text": "Export is available in CSV and JSON formats.",
        "sourceFactId": "f-3a1b22c0",
        "polishNote": "Removed redundant 'new'; tightened phrasing."
      }
    ],
    "refinedAt": "2026-06-28T10:00:05Z"
  },
  "document": {
    "title": "Product release notes: export feature",
    "summary": "The export feature now supports CSV and JSON, with a 10,000-row limit per batch request.",
    "sections": [
      {
        "sectionId": "export-capabilities",
        "heading": "Export capabilities",
        "body": "Users can now export data in CSV or JSON format. Batch exports are capped at 10,000 rows per request.",
        "citations": [
          { "refinedId": "r-3a1b22c0", "text": "Export is available in CSV and JSON formats." }
        ]
      }
    ],
    "formattedAt": "2026-06-28T10:00:10Z"
  },
  "quality": {
    "score": 5,
    "rationale": "Fact coverage, citation completeness, structural parity, and title presence all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:11Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:11Z",
  "guardrailRejections": [
    {
      "stage": "EXTRACT",
      "failedField": "facts",
      "reason": "output-validation-failure: facts list is empty; at least one Fact is required",
      "rejectedAt": "2026-06-28T10:00:01Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when the agent returned an invalid stage output and the guardrail caught it.

### SSE event format

```
event: document-update
data: { "documentId": "d-5af9c2...", "status": "REFINED", "refined": { ... }, ... }
```

One event per state transition (`CREATED`, `EXTRACTING`, `EXTRACTED`, `REFINING`, `REFINED`, `FORMATTING`, `FORMATTED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: document-rejection
data: { "documentId": "d-5af9c2...", "stage": "EXTRACT", "failedField": "facts", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `documentId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitDocumentRequest` record and the `DocumentCreated` event to carry it.
