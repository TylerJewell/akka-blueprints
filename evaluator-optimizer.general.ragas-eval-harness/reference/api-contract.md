# API contract — ragas-eval-harness

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `{ "question": String, "corpusTag"?: String, "submittedBy"?: String }` | `202 { "runId": String }` | `EvalEndpoint` → `QuestionQueue` |
| `GET` | `/api/runs` | — | `200 [ EvalRun... ]` (optional `?status=ANSWERING\|EVALUATING\|PASSED\|FAILED_FINAL` — filtered client-side from `getAllRuns`) | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/runs/{id}` | — | `200 EvalRun` / `404` | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` (one event per run change) | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/ci/gate-status` | — | `200 { "gatePassed": Boolean, "passRate": Double, "windowSize": Integer }` | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/runs`

- Missing `corpusTag` → `null` (no corpus filter).
- Missing `submittedBy` → `"anonymous"`.
- `question` must be non-empty and at most 2000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(question, submittedBy)`; the second submission returns the first `runId` (200) instead of starting a new workflow.

## JSON shapes

### EvalRun

```json
{
  "runId": "r-9c1a4…",
  "questionText": "What is the default retry ceiling in the evaluation harness?",
  "corpusTag": null,
  "maxAttempts": 3,
  "status": "PASSED",
  "attempts": [
    {
      "attemptNumber": 1,
      "answer": {
        "text": "The default retry ceiling is 3 attempts…",
        "retrievedChunks": [
          { "chunkId": "c-001", "sourceDoc": "spec.md", "text": "max-attempts = 3…", "relevanceScore": 0.91 }
        ],
        "groundingConfidence": 0.75,
        "answeredAt": "2026-06-28T09:01:02Z"
      },
      "grounding": { "passed": true, "reasonCode": "OK", "detail": "" },
      "score": {
        "verdict": "RETRY",
        "feedback": {
          "failedMetrics": ["faithfulness", "contextPrecision"],
          "metricScores": {
            "faithfulness": 0.58,
            "answerRelevance": 0.83,
            "contextPrecision": 0.62
          },
          "overallRationale": "Two sentences introduce claims not present in retrieved chunks."
        },
        "faithfulness": 0.58,
        "answerRelevance": 0.83,
        "contextPrecision": 0.62,
        "evaluatedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "answer": {
        "text": "The default retry ceiling is 3 attempts, configured via ragas-eval.workflow.max-attempts…",
        "retrievedChunks": [
          { "chunkId": "c-001", "sourceDoc": "spec.md", "text": "max-attempts = 3…", "relevanceScore": 0.94 },
          { "chunkId": "c-002", "sourceDoc": "spec.md", "text": "FAILED_FINAL preserves all attempts…", "relevanceScore": 0.88 }
        ],
        "groundingConfidence": 0.90,
        "answeredAt": "2026-06-28T09:01:21Z"
      },
      "grounding": { "passed": true, "reasonCode": "OK", "detail": "" },
      "score": {
        "verdict": "PASS",
        "feedback": {
          "failedMetrics": [],
          "metricScores": {
            "faithfulness": 0.92,
            "answerRelevance": 0.88,
            "contextPrecision": 0.82
          },
          "overallRationale": "All three metrics clear thresholds; answer is grounded and directly responsive."
        },
        "faithfulness": 0.92,
        "answerRelevance": 0.88,
        "contextPrecision": 0.82,
        "evaluatedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "passedAttemptNumber": 2,
  "passedAnswerText": "The default retry ceiling is 3 attempts…",
  "failureReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Grounding verdict forms

```json
{ "passed": true,  "reasonCode": "OK",                      "detail": "" }
{ "passed": false, "reasonCode": "BELOW_CONFIDENCE_FLOOR",  "detail": "Grounding confidence 0.28 is below floor 0.40." }
```

`reasonCode` is one of `OK`, `BELOW_CONFIDENCE_FLOOR`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### CI gate response

```json
{ "gatePassed": true,  "passRate": 0.87, "windowSize": 100 }
{ "gatePassed": false, "passRate": 0.73, "windowSize": 100 }
```

`windowSize` is the number of completed runs considered (up to 100). A CI script should exit non-zero when `gatePassed` is `false`.

### SSE event format

```
event: run-update
data: { "runId": "r-9c1a4…", "status": "EVALUATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `runId`. The full `EvalRun` JSON is included so a fresh client can render the row without a separate fetch.
