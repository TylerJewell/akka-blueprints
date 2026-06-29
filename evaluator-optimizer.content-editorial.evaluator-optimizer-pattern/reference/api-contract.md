# API contract — evaluator-optimizer-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/headlines` | `{ "summary": String, "wordCeiling"?: Integer, "requestedBy"?: String }` | `202 { "headlineId": String }` | `EditorialEndpoint` → `SubmissionQueue` |
| `GET` | `/api/headlines` | — | `200 [ Headline... ]` (optional `?status=DRAFTING\|REVIEWING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllHeadlines`) | `EditorialEndpoint` ← `HeadlinesView` |
| `GET` | `/api/headlines/{id}` | — | `200 Headline` / `404` | `EditorialEndpoint` ← `HeadlinesView` |
| `GET` | `/api/headlines/sse` | — | `text/event-stream` (one event per headline change) | `EditorialEndpoint` ← `HeadlinesView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/headlines`

- Missing `wordCeiling` → 12 (from `evaluator-optimizer-workflow.headline.default-word-ceiling`).
- Missing `requestedBy` → `"anonymous"`.
- `wordCeiling` must be in `[3, 30]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(summary, requestedBy)`; the second submission returns the first `headlineId` (200) instead of starting a new workflow.

## JSON shapes

### Headline

```json
{
  "headlineId": "h-3a7c1…",
  "summary": "Researchers found that daily 15-minute walks reduce cardiovascular risk in adults over sixty.",
  "wordCeiling": 12,
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "draft": {
        "text": "Researchers Find Daily Walks Reduce Heart Disease Risk in Older Adults",
        "wordCount": 12,
        "draftedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Lead noun \"Researchers\" is generic — name the finding instead.",
            "\"Reduce\" is weak; the summary gives a magnitude, use it.",
            "Passive construction in \"Risk in Older Adults\" — restructure."
          ],
          "overallRationale": "Specificity and word economy fall below threshold."
        },
        "score": 2,
        "reviewedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "draft": {
        "text": "Daily 15-Minute Walk Cuts Heart Disease Risk After 60",
        "wordCount": 10,
        "draftedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "APPROVE",
        "notes": { "bullets": [], "overallRationale": "Specific outcome named; active voice; clear and faithful to the summary." },
        "score": 5,
        "reviewedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedText": "Daily 15-Minute Walk Cuts Heart Disease Risk After 60",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",           "detail": "" }
{ "passed": false, "reasonCode": "OVER_CEILING",  "detail": "Draft is 14 words; ceiling is 12." }
```

`reasonCode` is one of `OK`, `OVER_CEILING`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: headline-update
data: { "headlineId": "h-3a7c1…", "status": "REVIEWING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `headlineId`. The full `Headline` JSON is included so a fresh client can render the row without a separate fetch.
