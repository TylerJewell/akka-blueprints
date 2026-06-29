# API contract — ab-model-eval

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/trials` | `{ "text": String, "preferredCandidate"?: String, "submittedBy"?: String }` | `202 { "trialId": String }` | `EvalEndpoint` → `TaskQueueEntity` |
| `GET` | `/api/trials` | — | `200 [ Trial... ]` (optional `?status=RUNNING\|JUDGED\|TIMED_OUT` — filtered client-side from `getAllTrials`) | `EvalEndpoint` ← `TrialsView` |
| `GET` | `/api/trials/{id}` | — | `200 Trial` / `404` | `EvalEndpoint` ← `TrialsView` |
| `GET` | `/api/trials/sse` | — | `text/event-stream` (one event per trial change) | `EvalEndpoint` ← `TrialsView` |
| `POST` | `/api/trials/promote` | — | `200 { "promoted": true }` / `409 { "reason": "recertification-required" }` | `EvalEndpoint` |
| `POST` | `/api/trials/recertify` | — | `200 { "cleared": Number }` (count of flagged trials cleared) | `EvalEndpoint` → `TrialEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/trials`

- Missing `preferredCandidate` → `"A"`.
- Missing `submittedBy` → `"anonymous"`.
- `text` must be non-empty and at most 2000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(text, submittedBy)`; the second submission returns the first `trialId` (200) instead of starting a new workflow.

## JSON shapes

### Trial

```json
{
  "trialId": "t-9c4b1…",
  "taskText": "What is the capital of France?",
  "preferredCandidate": "A",
  "timeoutSeconds": 60,
  "status": "JUDGED",
  "responseA": {
    "candidateId": "A",
    "text": "Paris. France's government and most national institutions are based there.",
    "tokensUsed": 18,
    "respondedAt": "2026-06-28T10:01:04Z"
  },
  "responseB": {
    "candidateId": "B",
    "text": "Paris. It has served as France's capital since the late 10th century, housing the national legislature…",
    "tokensUsed": 42,
    "respondedAt": "2026-06-28T10:01:05Z"
  },
  "judgement": {
    "winner": "A",
    "scoresA": { "accuracy": 5, "relevance": 5, "clarity": 5, "conciseness": 5 },
    "scoresB": { "accuracy": 5, "relevance": 4, "clarity": 5, "conciseness": 3 },
    "totalA": 20,
    "totalB": 17,
    "rationale": "Candidate A is maximally accurate and concise; candidate B's additional context reduces relevance and conciseness scores.",
    "judgedAt": "2026-06-28T10:01:11Z"
  },
  "recertificationRequired": false,
  "failureReason": null,
  "createdAt": "2026-06-28T10:01:00Z",
  "finishedAt": "2026-06-28T10:01:12Z"
}
```

### Timed-out trial

```json
{
  "trialId": "t-2f8e3…",
  "taskText": "Explain gradient descent.",
  "status": "TIMED_OUT",
  "responseA": null,
  "responseB": null,
  "judgement": null,
  "recertificationRequired": false,
  "failureReason": "candidateAStep exceeded stepTimeout(60s) after 1 retry",
  "createdAt": "2026-06-28T10:05:00Z",
  "finishedAt": "2026-06-28T10:06:02Z"
}
```

### Recertification 409

```json
{
  "error": "recertification-required",
  "detail": "Preferred candidate win-rate is 0.45 over the last 20 trials; threshold is 0.60. Call POST /api/trials/recertify to acknowledge and re-enable promotion."
}
```

### SSE event format

```
event: trial-update
data: { "trialId": "t-9c4b1…", "status": "JUDGED", "responseA": {...}, "responseB": {...}, "judgement": {...}, ... }
```

One event per state transition. Clients reconcile by `trialId`. The full `Trial` JSON is included so a fresh client can render the row without a separate fetch.
