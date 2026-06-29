# API contract — seo-rubric-critic

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/articles` | `{ "title": String, "bodyText": String, "targetKeyword": String, "submittedBy"?: String }` | `202 { "articleId": String }` | `AuditEndpoint` → `SubmissionQueue` |
| `GET` | `/api/articles` | — | `200 [ Article... ]` (optional `?status=SCORING\|REVISING\|APPROVED\|FAILED_FINAL` — filtered client-side from `getAllArticles`) | `AuditEndpoint` ← `ArticlesView` |
| `GET` | `/api/articles/{id}` | — | `200 Article` / `404` | `AuditEndpoint` ← `ArticlesView` |
| `GET` | `/api/articles/sse` | — | `text/event-stream` (one event per article change) | `AuditEndpoint` ← `ArticlesView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/articles`

- Missing `submittedBy` → `"anonymous"`.
- `wordCountCeiling` defaults to 1500 (from `seo-rubric-critic.audit.default-word-count-ceiling`); overridable per-article by including it in the body.
- `maxRounds` defaults to 4 (from `seo-rubric-critic.audit.max-rounds`).
- `acceptanceThreshold` defaults to 75 (from `seo-rubric-critic.audit.acceptance-threshold`).
- `bodyText` must be at least 50 words; otherwise `400`.
- Duplicate-detection window: 10 s on `(title, submittedBy)`; the second submission returns the first `articleId` (200) instead of starting a new workflow.

## JSON shapes

### Article

```json
{
  "articleId": "a-9e3b1…",
  "title": "How to Write an SEO-Optimised Blog Post in 2026",
  "targetKeyword": "SEO blog post",
  "wordCountCeiling": 1500,
  "maxRounds": 4,
  "acceptanceThreshold": 75,
  "status": "APPROVED",
  "rounds": [
    {
      "roundNumber": 1,
      "draft": {
        "title": "How to Write an SEO-Optimised Blog Post in 2026",
        "bodyText": "SEO has changed dramatically…",
        "wordCount": 1120,
        "draftedAt": "2026-06-28T10:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "rubricScore": {
        "passed": false,
        "compositeScore": 57,
        "dimensions": [
          { "dimension": "Depth", "score": 2, "note": "Missing the 'how long does it take' sub-query." },
          { "dimension": "Keyword placement", "score": 2, "note": "Keyword absent from first 100 words." }
        ],
        "feedback": {
          "failingDimensions": [
            { "dimension": "Depth", "score": 2, "note": "Missing sub-query coverage." },
            { "dimension": "Keyword placement", "score": 2, "note": "Keyword absent from opening." }
          ],
          "improvementSummary": "Add a 'Time to results' section and move the keyword into the opening paragraph."
        },
        "scoredAt": "2026-06-28T10:01:14Z"
      }
    },
    {
      "roundNumber": 2,
      "draft": {
        "title": "How to Write an SEO-Optimised Blog Post in 2026",
        "bodyText": "Writing an SEO blog post that ranks…",
        "wordCount": 1290,
        "draftedAt": "2026-06-28T10:01:33Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "rubricScore": {
        "passed": true,
        "compositeScore": 81,
        "dimensions": [],
        "feedback": {
          "failingDimensions": [],
          "improvementSummary": "All mandatory dimensions pass. Depth and keyword placement both at 4."
        },
        "scoredAt": "2026-06-28T10:01:47Z"
      }
    }
  ],
  "approvedAtRound": 2,
  "approvedBodyText": "Writing an SEO blog post that ranks…",
  "rejectionReason": null,
  "createdAt": "2026-06-28T10:00:59Z",
  "finishedAt": "2026-06-28T10:01:48Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "OVER_WORD_CEILING", "detail": "Draft is 1620 words; ceiling is 1500." }
```

`reasonCode` is one of `OK`, `OVER_WORD_CEILING`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: article-update
data: { "articleId": "a-9e3b1…", "status": "REVISING", "rounds": [...], ... }
```

One event per state transition. Clients reconcile by `articleId`. The full `Article` JSON is included so a fresh client can render the row without a separate fetch.
