# API contract — generator-critic

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/essays` | `{ "topic": String, "wordCeiling"?: Integer, "requestedBy"?: String }` | `202 { "essayId": String }` | `EssayEndpoint` → `SubmissionQueue` |
| `GET` | `/api/essays` | — | `200 [ Essay... ]` (optional `?status=DRAFTING\|REFLECTING\|ACCEPTED\|RELEASED\|REJECTED_FINAL` — filtered client-side from `getAllEssays`) | `EssayEndpoint` ← `EssaysView` |
| `GET` | `/api/essays/{id}` | — | `200 Essay` / `404` | `EssayEndpoint` ← `EssaysView` |
| `GET` | `/api/essays/sse` | — | `text/event-stream` (one event per essay change) | `EssayEndpoint` ← `EssaysView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/essays`

- Missing `wordCeiling` → 400 (from `basic-reflection.reflection.default-word-ceiling`).
- Missing `requestedBy` → `"anonymous"`.
- `wordCeiling` must be in `[50, 3000]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(topic, requestedBy)`; the second submission returns the first `essayId` (200) instead of starting a new workflow.

## JSON shapes

### Essay

```json
{
  "essayId": "e-4b9c1…",
  "topic": "the long-term effects of remote work on urban planning",
  "wordCeiling": 400,
  "maxRounds": 4,
  "status": "RELEASED",
  "rounds": [
    {
      "roundNumber": 1,
      "draft": {
        "text": "Remote work, once a niche arrangement, …",
        "wordCount": 378,
        "draftedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "reflection": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Paragraph 3 asserts a causal link without evidence; qualify or remove.",
            "Conclusion introduces a new idea not developed in the body.",
            "Paragraph 2 contradicts the thesis in paragraph 1."
          ],
          "overallRationale": "Coherence and factual plausibility fall below threshold."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "roundNumber": 2,
      "draft": {
        "text": "Remote work, once a niche arrangement, …",
        "wordCount": 362,
        "draftedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "reflection": {
        "verdict": "ACCEPT",
        "notes": { "bullets": [], "overallRationale": "Thesis clear; body paragraphs develop it without drift; conclusion grounded." },
        "score": 5,
        "evaluatedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "acceptedRoundNumber": 2,
  "releasedText": "Remote work, once a niche arrangement, …",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:38Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "POLICY_VIOLATION",  "detail": "Draft contains flagged phrase: \"[example phrase]\"." }
```

`reasonCode` is one of `OK`, `POLICY_VIOLATION`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: essay-update
data: { "essayId": "e-4b9c1…", "status": "REFLECTING", "rounds": [...], ... }
```

One event per state transition. Clients reconcile by `essayId`. The full `Essay` JSON is included so a fresh client can render the row without a separate fetch.
