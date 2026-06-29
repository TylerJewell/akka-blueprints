# API contract — llm-judge-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/evaluations` | `{ "questionText": String, "domainTag"?: String, "scoreThreshold"?: Integer, "submittedBy"?: String }` | `202 { "evaluationId": String }` | `EvaluationEndpoint` → `QuestionQueue` |
| `GET` | `/api/evaluations` | — | `200 [ Evaluation... ]` (optional `?status=GENERATING\|JUDGING\|ACCEPTED\|REJECTED_FINAL` — filtered client-side from `getAllEvaluations`) | `EvaluationEndpoint` ← `EvaluationsView` |
| `GET` | `/api/evaluations/{id}` | — | `200 Evaluation` / `404` | `EvaluationEndpoint` ← `EvaluationsView` |
| `GET` | `/api/evaluations/sse` | — | `text/event-stream` (one event per evaluation change) | `EvaluationEndpoint` ← `EvaluationsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/evaluations`

- Missing `domainTag` → `"general"`.
- Missing `scoreThreshold` → `4` (from `llm-judge-loop.evaluation.default-score-threshold`).
- Missing `submittedBy` → `"anonymous"`.
- `scoreThreshold` must be in `[1, 5]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(questionText, submittedBy)`; the second submission returns the first `evaluationId` (200) instead of starting a new workflow.

## JSON shapes

### Evaluation

```json
{
  "evaluationId": "ev-3c8b1…",
  "questionText": "What causes the aurora borealis?",
  "domainTag": "science",
  "scoreThreshold": 4,
  "maxAttempts": 4,
  "status": "ACCEPTED",
  "attempts": [
    {
      "attemptNumber": 1,
      "answer": {
        "text": "The aurora borealis results from interactions between charged particles...",
        "tokenCount": 142,
        "generatedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "judgment": {
        "verdict": "REVISE",
        "feedback": {
          "bullets": [
            "The altitude range claim lacks observational basis.",
            "Paragraph 2 introduces CMEs without connecting them to the prior mechanism.",
            "Final sentence speculates without grounding in the question."
          ],
          "overallRationale": "Accuracy and coherence fall below threshold despite sound completeness."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "answer": {
        "text": "The aurora borealis results from interactions between charged particles...",
        "tokenCount": 178,
        "generatedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "judgment": {
        "verdict": "ACCEPT",
        "feedback": { "bullets": [], "overallRationale": "All four dimensions at threshold; answer is grounded and coherent." },
        "score": 5,
        "evaluatedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "acceptedAttemptNumber": 2,
  "acceptedText": "The aurora borealis results from interactions between charged particles...",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",              "detail": "" }
{ "passed": false, "reasonCode": "STRUCTURAL_FAIL",  "detail": "Answer token count is 23; minimum is 50." }
```

`reasonCode` is one of `OK`, `STRUCTURAL_FAIL`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: evaluation-update
data: { "evaluationId": "ev-3c8b1…", "status": "JUDGING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `evaluationId`. The full `Evaluation` JSON is included so a fresh client can render the row without a separate fetch.
