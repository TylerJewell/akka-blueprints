# API contract — self-eval-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/posts` | `{ "topic": String, "characterCeiling"?: Integer, "requestedBy"?: String }` | `202 { "postId": String }` | `PoetryEndpoint` → `RequestQueue` |
| `GET` | `/api/posts` | — | `200 [ Post... ]` (optional `?status=DRAFTING\|EVALUATING\|ACCEPTED\|REJECTED_FINAL` — filtered client-side from `getAllPosts`) | `PoetryEndpoint` ← `PostsView` |
| `GET` | `/api/posts/{id}` | — | `200 Post` / `404` | `PoetryEndpoint` ← `PostsView` |
| `GET` | `/api/posts/sse` | — | `text/event-stream` (one event per post change) | `PoetryEndpoint` ← `PostsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/posts`

- Missing `characterCeiling` → 280 (from `self-eval-loop.refinement.default-character-ceiling`).
- Missing `requestedBy` → `"anonymous"`.
- `characterCeiling` must be in `[40, 2000]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(topic, requestedBy)`; the second submission returns the first `postId` (200) instead of starting a new workflow.

## JSON shapes

### Post

```json
{
  "postId": "p-7f2a3…",
  "topic": "the first frost of October on a city park",
  "characterCeiling": 280,
  "maxAttempts": 4,
  "status": "ACCEPTED",
  "attempts": [
    {
      "attemptNumber": 1,
      "draft": {
        "text": "Soft creeps the rime upon yon iron gate, …",
        "characterCount": 268,
        "draftedAt": "2026-06-28T08:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "critique": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Line 3 introduces \"traffic\", which is not Elizabethan.",
            "Closing couplet abandons the frost imagery.",
            "Metre breaks in line 5."
          ],
          "overallRationale": "Register and consistency fall below threshold."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T08:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "draft": {
        "text": "Soft creeps the rime upon yon iron gate, …",
        "characterCount": 264,
        "draftedAt": "2026-06-28T08:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "critique": {
        "verdict": "ACCEPT",
        "notes": { "bullets": [], "overallRationale": "Register holds; imagery period-consistent." },
        "score": 5,
        "evaluatedAt": "2026-06-28T08:01:33Z"
      }
    }
  ],
  "acceptedAttemptNumber": 2,
  "acceptedText": "Soft creeps the rime upon yon iron gate, …",
  "rejectionReason": null,
  "createdAt": "2026-06-28T08:00:59Z",
  "finishedAt": "2026-06-28T08:01:34Z"
}
```

### Guardrail verdict form

```json
{ "passed": true,  "reasonCode": "OK",            "detail": "" }
{ "passed": false, "reasonCode": "OVER_CEILING",  "detail": "Draft is 312 characters; ceiling is 280." }
```

`reasonCode` is one of `OK`, `OVER_CEILING`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: post-update
data: { "postId": "p-7f2a3…", "status": "EVALUATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `postId`. The full `Post` JSON is included so a fresh client can render the row without a separate fetch.
