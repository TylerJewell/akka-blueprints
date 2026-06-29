# API contract — writer-reviewer-doc-gen

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/documents` | `{ "topic": String, "wordCeiling"?: Integer, "requestedBy"?: String }` | `202 { "documentId": String }` | `DocumentEndpoint` → `RequestQueue` |
| `GET` | `/api/documents` | — | `200 [ Document... ]` (optional `?status=DRAFTING\|REVIEWING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllDocuments`) | `DocumentEndpoint` ← `DocumentsView` |
| `GET` | `/api/documents/{id}` | — | `200 Document` / `404` | `DocumentEndpoint` ← `DocumentsView` |
| `GET` | `/api/documents/sse` | — | `text/event-stream` (one event per document change) | `DocumentEndpoint` ← `DocumentsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/documents`

- Missing `wordCeiling` → 500 (from `writer-reviewer-doc-gen.refinement.default-word-ceiling`).
- Missing `requestedBy` → `"anonymous"`.
- `wordCeiling` must be in `[50, 5000]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(topic, requestedBy)`; the second submission returns the first `documentId` (200) instead of starting a new workflow.

## JSON shapes

### Document

```json
{
  "documentId": "d-8a3f1…",
  "topic": "the role of open-source software in cloud infrastructure",
  "wordCeiling": 500,
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "draft": {
        "text": "Open-source software forms the operational substrate…",
        "wordCount": 488,
        "draftedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Section 2 introduces \"CNCF\" without definition.",
            "Conclusion restates the introduction verbatim.",
            "Section 3 final paragraph trails off without closure."
          ],
          "overallRationale": "Clarity and structure fall below threshold despite sound length and topic adherence."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "draft": {
        "text": "Open-source software forms the operational substrate…",
        "wordCount": 467,
        "draftedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "APPROVE",
        "notes": { "bullets": [], "overallRationale": "Structure sound; prose clear; topic addressed throughout." },
        "score": 5,
        "evaluatedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedText": "Open-source software forms the operational substrate…",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Guardrail verdict form

```json
{ "passed": true,  "reasonCode": "OK",            "detail": "" }
{ "passed": false, "reasonCode": "OVER_CEILING",  "detail": "Draft is 537 words; ceiling is 500." }
```

`reasonCode` is one of `OK`, `OVER_CEILING`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: document-update
data: { "documentId": "d-8a3f1…", "status": "REVIEWING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `documentId`. The full `Document` JSON is included so a fresh client can render the row without a separate fetch.
