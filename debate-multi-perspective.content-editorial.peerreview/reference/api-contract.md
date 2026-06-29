# API contract — peerreview

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/reviews` | `{ "title": String, "body": String, "submittedBy"?: String }` | `202 { "reviewId": String }` | `ReviewEndpoint` → `SubmissionQueue` |
| `GET` | `/api/reviews` | — | `200 [ Review... ]` | `ReviewEndpoint` ← `ReviewView` |
| `GET` | `/api/reviews?status=SYNTHESISED` | — | `200 [ Review... ]` (filtered client-side) | `ReviewEndpoint` ← `ReviewView` |
| `GET` | `/api/reviews/{id}` | — | `200 Review` / `404` | `ReviewEndpoint` ← `ReviewView` |
| `GET` | `/api/reviews/sse` | — | `text/event-stream` (one event per review change) | `ReviewEndpoint` ← `ReviewView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

The submitted `body` is consumed by the workflow and redacted at intake; it is never returned by any `GET` endpoint. Only the redacted body and the review findings are ever read back.

## JSON shapes

### Review

```json
{
  "reviewId": "rv-7f2…",
  "title": "Q3 product-launch blog post",
  "status": "SYNTHESISED",
  "redactedBody": "Contact [NAME] at [EMAIL] … the launch …",
  "redactionCount": 3,
  "technical": {
    "axis": "TECHNICAL",
    "verdict": "PASS",
    "score": 4,
    "findings": [ { "severity": "MINOR", "comment": "Benchmark figure lacks a unit." } ],
    "reviewedAt": "2026-06-28T07:33:01Z"
  },
  "style": {
    "axis": "STYLE",
    "verdict": "REVISE",
    "score": 3,
    "findings": [ { "severity": "MAJOR", "comment": "The second section buries the lede." } ],
    "reviewedAt": "2026-06-28T07:33:03Z"
  },
  "compliance": {
    "axis": "COMPLIANCE",
    "verdict": "PASS",
    "score": 4,
    "findings": [ { "severity": "INFO", "comment": "No disclosure obligation triggered." } ],
    "reviewedAt": "2026-06-28T07:33:04Z"
  },
  "verdict": {
    "verdict": "REVISE",
    "summary": "Two axes pass; the style axis raised a major structural issue that caps the verdict at REVISE …",
    "axisReviews": [ { "axis": "TECHNICAL" }, { "axis": "STYLE" }, { "axis": "COMPLIANCE" } ],
    "guardrailVerdict": "ok",
    "synthesisedAt": "2026-06-28T07:33:09Z"
  },
  "failureReason": null,
  "consistencyScore": 5,
  "consistencyRationale": "Verdict severity tracks the single MAJOR style finding.",
  "createdAt": "2026-06-28T07:32:55Z",
  "finishedAt": "2026-06-28T07:33:09Z"
}
```

Lifecycle fields (`redactedBody`, `redactionCount`, `technical`, `style`, `compliance`, `verdict`, `failureReason`, `consistencyScore`, `consistencyRationale`, `finishedAt`) are `Optional<T>` in Java and serialize as the raw value or `null` — never an `{ "value": … }` wrapper (Lesson 6).

### SSE event format

```
event: review-update
data: { "reviewId": "rv-7f2…", "status": "REVIEWING", ... }
```

One event per state transition. Clients reconcile by `reviewId`. The `ConsistencyScored` transition emits a `review-update` event whose status is unchanged but whose `consistencyScore` is now populated.
