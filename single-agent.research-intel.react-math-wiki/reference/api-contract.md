# API contract — react-math-wiki

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/questions` | `SubmitQuestionRequest` | `201 { questionId }` | `ResearchEndpoint` → `ResearchEntity` |
| `GET` | `/api/questions` | — | `200 [ QuestionRow... ]` (newest-first) | `ResearchEndpoint` ← `ResearchView` |
| `GET` | `/api/questions/{id}` | — | `200 QuestionRow` / `404` | `ResearchEndpoint` ← `ResearchView` |
| `GET` | `/api/questions/sse` | — | `text/event-stream` | `ResearchEndpoint` ← `ResearchView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQuestionRequest (request body)

```json
{
  "text": "What year was the Eiffel Tower completed, and how many years ago was that?",
  "submittedBy": "user-42"
}
```

### QuestionRow (response body)

```json
{
  "questionId": "q-8a3f...",
  "question": {
    "questionId": "q-8a3f...",
    "text": "What year was the Eiffel Tower completed, and how many years ago was that?",
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "answer": {
    "answerText": "The Eiffel Tower was completed in 1889, which is 137 years ago.",
    "confidence": "HIGH",
    "trace": [
      {
        "step": 1,
        "tool": "SEARCH_WIKIPEDIA",
        "argument": "Eiffel Tower history completion year",
        "observation": "The Eiffel Tower was built from 1887 to 1889 as the centerpiece of the 1889 World's Fair."
      },
      {
        "step": 2,
        "tool": "EVALUATE_MATH",
        "argument": "2026 - 1889",
        "observation": "137.0"
      }
    ],
    "stepCount": 2,
    "answeredAt": "2026-06-28T14:00:18Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Answer references the retrieved passage and the math result is consistent with the trace.",
    "evaluatedAt": "2026-06-28T14:00:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: question-update
data: { "questionId": "q-8a3f...", "status": "RUNNING", "answer": null, ... }

event: question-update
data: { "questionId": "q-8a3f...", "status": "RUNNING", "answer": null, "trace": [{"step":1,...}], ... }

event: question-update
data: { "questionId": "q-8a3f...", "status": "EVALUATED", "answer": { ... }, "eval": { ... }, ... }
```

One event fires per state transition (`SUBMITTED`, `RUNNING`) and per `ToolCallRecorded` event (while in `RUNNING`), plus the terminal transitions (`ANSWERED`, `EVALUATED`, `FAILED`). Each event carries the full row at that moment, so a late-joining client never needs to replay. Clients reconcile by `questionId`.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
