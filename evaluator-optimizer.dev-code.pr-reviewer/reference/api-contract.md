# API contract — pr-reviewer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/reviews` | `{ "diffText": String, "description"?: String, "submittedBy"?: String }` | `202 { "reviewId": String }` | `ReviewEndpoint` → `PrQueue` |
| `GET` | `/api/reviews` | — | `200 [ Review... ]` (optional `?status=REVIEWING\|CHECKING_ALIGNMENT\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllReviews`) | `ReviewEndpoint` ← `ReviewsView` |
| `GET` | `/api/reviews/{id}` | — | `200 Review` / `404` | `ReviewEndpoint` ← `ReviewsView` |
| `GET` | `/api/reviews/sse` | — | `text/event-stream` (one event per review change) | `ReviewEndpoint` ← `ReviewsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/reviews`

- Missing `description` → `""`.
- Missing `submittedBy` → `"anonymous"`.
- `diffText` must be non-empty and at most 50,000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(sha256(diffText), submittedBy)`; the second submission returns the first `reviewId` (200) instead of starting a new workflow.

## JSON shapes

### Review

```json
{
  "reviewId": "rv-9a1b2…",
  "diffText": "diff --git a/src/auth/TokenValidator.java …",
  "description": "Fix token expiry check",
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "draft": {
        "comments": [
          {
            "filePath": "src/auth/TokenValidator.java",
            "lineNumber": 78,
            "severity": "BLOCKER",
            "body": "The expiry check uses == rather than .equals(); expired tokens will always pass on non-interned Instant instances."
          }
        ],
        "commentCount": 1,
        "draftedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "alignmentCheck": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Comment lacks reference to API contract §4.2 to anchor the requirement.",
            "No SUGGESTION-level comments despite multiple style opportunities in the diff."
          ],
          "overallRationale": "Actionability falls below threshold; doc anchor missing."
        },
        "score": 3,
        "checkedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "draft": {
        "comments": [
          {
            "filePath": "src/auth/TokenValidator.java",
            "lineNumber": 78,
            "severity": "BLOCKER",
            "body": "The expiry check uses == rather than .equals() (see API contract §4.2, \"Token lifecycle\"); expired tokens will always pass validation on non-interned Instant instances."
          },
          {
            "filePath": "src/auth/TokenValidator.java",
            "lineNumber": 92,
            "severity": "SUGGESTION",
            "body": "Extracting the clock dependency into a field would make this class easier to test against a fixed Instant."
          }
        ],
        "commentCount": 2,
        "draftedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "alignmentCheck": {
        "verdict": "APPROVE",
        "notes": {
          "bullets": [],
          "overallRationale": "All comments are impersonal, references match the diff, severity is calibrated, and each comment is independently actionable."
        },
        "score": 5,
        "checkedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedComments": [
    {
      "filePath": "src/auth/TokenValidator.java",
      "lineNumber": 78,
      "severity": "BLOCKER",
      "body": "The expiry check uses == rather than .equals() …"
    }
  ],
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "PERSONAL_CRITIQUE", "detail": "Comment at src/auth/TokenValidator.java:78 contains 'you forgot to', which is personal critique directed at the author." }
```

`reasonCode` is one of `OK`, `PERSONAL_CRITIQUE`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: review-update
data: { "reviewId": "rv-9a1b2…", "status": "CHECKING_ALIGNMENT", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `reviewId`. The full `Review` JSON is included so a fresh client can render the row without a separate fetch.
