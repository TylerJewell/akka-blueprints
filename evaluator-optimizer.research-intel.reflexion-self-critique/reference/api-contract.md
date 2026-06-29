# API contract — reflexion-self-critique

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `{ "questionText": String, "citationFloor"?: Integer, "submittedBy"?: String }` | `202 { "queryId": String }` | `ResearchEndpoint` → `QueryQueue` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (optional `?status=RESEARCHING\|REFLECTING\|RESOLVED\|EXHAUSTED` — filtered client-side from `getAllQueries`) | `ResearchEndpoint` ← `QueriesView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `ResearchEndpoint` ← `QueriesView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` (one event per query change) | `ResearchEndpoint` ← `QueriesView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/queries`

- Missing `citationFloor` → 2 (from `reflexion.default-citation-floor`).
- Missing `submittedBy` → `"anonymous"`.
- `citationFloor` must be in `[1, 20]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(questionText, submittedBy)`; the second submission returns the first `queryId` (200) instead of starting a new workflow.

## JSON shapes

### Query

```json
{
  "queryId": "q-4a1b9…",
  "questionText": "What are the main regulatory approaches to AI transparency in the EU?",
  "citationFloor": 2,
  "maxAttempts": 4,
  "status": "RESOLVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "answer": {
        "text": "EU AI transparency requirements operate at three levels...",
        "citations": [
          { "docId": "eu-ai-act-art13", "title": "EU AI Act Article 13", "snippet": "...", "url": "..." },
          { "docId": "gdpr-art22", "title": "GDPR Article 22", "snippet": "...", "url": "..." }
        ],
        "citationCount": 2,
        "answeredAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "reflexion": {
        "verdict": "RETRY",
        "note": {
          "reinforcementParagraph": "The actor covered the foundational regulation but missed the 2023 corrigendum...",
          "focusBullets": [
            "Coverage of the 2023 AI Act corrigendum is absent.",
            "Source 'reuters-aiact-summary' is secondary; replace with the Official Journal.",
            "Claim about ESMA scope is unsupported."
          ],
          "overallRationale": "Coverage and source quality fall below threshold."
        },
        "score": 2,
        "reflectedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "answer": {
        "text": "EU AI transparency requirements operate at three levels... The 2023 corrigendum...",
        "citations": [
          { "docId": "eu-ai-act-art13", "title": "EU AI Act Article 13", "snippet": "...", "url": "..." },
          { "docId": "gdpr-art22", "title": "GDPR Article 22", "snippet": "...", "url": "..." },
          { "docId": "ai-act-corrigendum-2023", "title": "AI Act Corrigendum 2023", "snippet": "...", "url": "..." },
          { "docId": "esma-algo-2023", "title": "ESMA Algo Guidance 2023", "snippet": "...", "url": "..." }
        ],
        "citationCount": 4,
        "answeredAt": "2026-06-28T09:01:28Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "reflexion": {
        "verdict": "PASS",
        "note": {
          "reinforcementParagraph": "The actor retrieved four relevant primary sources and addressed the 2023 corrigendum...",
          "focusBullets": [],
          "overallRationale": "All four dimensions score at threshold."
        },
        "score": 5,
        "reflectedAt": "2026-06-28T09:01:41Z"
      }
    }
  ],
  "resolvedAttemptNumber": 2,
  "resolvedAnswerText": "EU AI transparency requirements operate at three levels...",
  "exhaustionReason": null,
  "createdAt": "2026-06-28T09:00:58Z",
  "finishedAt": "2026-06-28T09:01:42Z"
}
```

### Citation guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",         "detail": "" }
{ "passed": false, "reasonCode": "UNDER_CITED", "detail": "Found 1 citation; minimum is 2." }
```

`reasonCode` is one of `OK`, `UNDER_CITED`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: query-update
data: { "queryId": "q-4a1b9…", "status": "REFLECTING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `queryId`. The full `Query` JSON is included so a fresh client can render the row without a separate fetch.
