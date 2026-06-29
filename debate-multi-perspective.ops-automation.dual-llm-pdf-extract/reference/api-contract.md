# API contract — dual-llm-pdf-extract

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/extractions` | `{ "filename": String, "extractedText": String, "submittedBy"?: String }` | `202 { "extractionId": String }` | `ExtractionEndpoint` → `DocumentQueue` |
| `GET` | `/api/extractions` | — | `200 [ Extraction... ]` | `ExtractionEndpoint` ← `ExtractionView` |
| `GET` | `/api/extractions?status=RECONCILED` | — | `200 [ Extraction... ]` (filtered client-side) | `ExtractionEndpoint` ← `ExtractionView` |
| `GET` | `/api/extractions/{id}` | — | `200 Extraction` / `404` | `ExtractionEndpoint` ← `ExtractionView` |
| `GET` | `/api/extractions/sse` | — | `text/event-stream` (one event per extraction change) | `ExtractionEndpoint` ← `ExtractionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

The submitted `extractedText` is consumed by the workflow and redacted at intake; it is never returned by any `GET` endpoint. Only the redacted text and the extracted fields are ever read back.

## JSON shapes

### Extraction

```json
{
  "extractionId": "ex-9a3…",
  "filename": "invoice-2026-Q2.pdf",
  "status": "RECONCILED",
  "redactedText": "Invoice from [NAME] ([EMAIL]) to [NAME] …",
  "redactionCount": 4,
  "claudeExtraction": {
    "modelFamily": "CLAUDE",
    "fields": [
      { "name": "document_number", "value": "INV-00421", "confidence": 0.99 },
      { "name": "total_amount", "value": "12,450.00", "confidence": 0.97 },
      { "name": "document_date", "value": "2026-06-15", "confidence": 0.95 }
    ],
    "extractorNotes": "Issuer name was redacted; all numeric fields were clear.",
    "extractedAt": "2026-06-28T09:11:02Z"
  },
  "geminiExtraction": {
    "modelFamily": "GEMINI",
    "fields": [
      { "name": "document_number", "value": "INV-00421", "confidence": 0.98 },
      { "name": "total_amount", "value": "12450.00", "confidence": 0.96 },
      { "name": "document_date", "value": "15 June 2026", "confidence": 0.93 }
    ],
    "extractorNotes": "Date formatted differently from Claude; amounts match after normalization.",
    "extractedAt": "2026-06-28T09:11:04Z"
  },
  "mergedExtraction": {
    "fields": [
      { "name": "document_number", "value": "INV-00421", "confidence": 0.99 },
      { "name": "total_amount", "value": "12,450.00", "confidence": 0.97 },
      { "name": "document_date", "value": "2026-06-15", "confidence": 0.95 }
    ],
    "confidenceMap": {
      "document_number": 0.99,
      "total_amount": 0.97,
      "document_date": 0.95
    },
    "disagreementFields": [
      {
        "fieldName": "document_date",
        "agreed": false,
        "claudeValue": "2026-06-15",
        "geminiValue": "15 June 2026"
      }
    ],
    "reconciliationNotes": "Two of three fields agreed exactly. The date field shows a formatting difference only; semantic values are equivalent.",
    "disagreementCount": 1,
    "reconciledAt": "2026-06-28T09:11:09Z"
  },
  "failureReason": null,
  "agreementScore": 4,
  "agreementRationale": "Models agreed on all numeric fields; date format differed but values were semantically identical.",
  "createdAt": "2026-06-28T09:10:58Z",
  "finishedAt": "2026-06-28T09:11:09Z"
}
```

Lifecycle fields (`redactedText`, `redactionCount`, `claudeExtraction`, `geminiExtraction`, `mergedExtraction`, `failureReason`, `agreementScore`, `agreementRationale`, `finishedAt`) are `Optional<T>` in Java and serialize as the raw value or `null` — never an `{ "value": … }` wrapper (Lesson 6).

### SSE event format

```
event: extraction-update
data: { "extractionId": "ex-9a3…", "status": "EXTRACTING", ... }
```

One event per state transition. Clients reconcile by `extractionId`. The `AgreementScored` transition emits an `extraction-update` event whose status is unchanged but whose `agreementScore` is now populated.
