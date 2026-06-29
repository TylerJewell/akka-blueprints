# API contract — evaluator-optimizer-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "description": String, "tokenCeiling"?: Integer, "submittedBy"?: String }` | `202 { "jobId": String }` | `OptimizationEndpoint` → `SubmissionQueue` |
| `GET` | `/api/jobs` | — | `200 [ Job... ]` (optional `?status=GENERATING\|EVALUATING\|ACCEPTED\|REJECTED_FINAL` — filtered client-side from `getAllJobs`) | `OptimizationEndpoint` ← `JobsView` |
| `GET` | `/api/jobs/{id}` | — | `200 Job` / `404` | `OptimizationEndpoint` ← `JobsView` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` (one event per job change) | `OptimizationEndpoint` ← `JobsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/jobs`

- Missing `tokenCeiling` → 500 (from `evaluator-optimizer.loop.default-token-ceiling`).
- Missing `submittedBy` → `"anonymous"`.
- `tokenCeiling` must be in `[50, 4000]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(description, submittedBy)`; the second submission returns the first `jobId` (200) instead of starting a new workflow.

## JSON shapes

### Job

```json
{
  "jobId": "j-8e3c1…",
  "description": "Explain why exponential backoff is preferred over fixed-interval retry in distributed systems.",
  "tokenCeiling": 500,
  "maxAttempts": 4,
  "status": "ACCEPTED",
  "attempts": [
    {
      "attemptNumber": 1,
      "candidate": {
        "text": "Exponential backoff spaces retries at increasing intervals…",
        "tokenCount": 487,
        "generatedAt": "2026-06-28T10:01:04Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "evaluation": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Completeness: the error-handling section does not distinguish retryable from non-retryable errors.",
            "Accuracy: the claim in paragraph 2 omits the condition under which jitter is unnecessary.",
            "Conciseness: the final paragraph restates the opening without adding content."
          ],
          "overallRationale": "Accuracy and completeness fall below threshold despite adequate relevance and structure."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T10:01:17Z"
      }
    },
    {
      "attemptNumber": 2,
      "candidate": {
        "text": "Exponential backoff spaces retries at increasing intervals…",
        "tokenCount": 451,
        "generatedAt": "2026-06-28T10:01:24Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "evaluation": {
        "verdict": "ACCEPT",
        "notes": { "bullets": [], "overallRationale": "All four dimensions score 4 or above; accurate, tight, and complete." },
        "score": 5,
        "evaluatedAt": "2026-06-28T10:01:36Z"
      }
    }
  ],
  "acceptedAttemptNumber": 2,
  "acceptedText": "Exponential backoff spaces retries at increasing intervals…",
  "rejectionReason": null,
  "createdAt": "2026-06-28T10:01:01Z",
  "finishedAt": "2026-06-28T10:01:37Z"
}
```

### Guardrail verdict form

```json
{ "passed": true,  "reasonCode": "OK",            "detail": "" }
{ "passed": false, "reasonCode": "OVER_CEILING",  "detail": "Candidate is 562 tokens; ceiling is 500." }
```

`reasonCode` is one of `OK`, `OVER_CEILING`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: job-update
data: { "jobId": "j-8e3c1…", "status": "EVALUATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `jobId`. The full `Job` JSON is included so a fresh client can render the row without a separate fetch.
