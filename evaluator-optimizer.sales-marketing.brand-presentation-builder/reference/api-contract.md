# API contract — brand-presentation-builder

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/presentations` | `{ "topic": String, "targetAudience"?: String, "slideCount"?: Integer, "wordsPerSlide"?: Integer, "requestedBy"?: String }` | `202 { "presentationId": String }` | `PresentationEndpoint` → `RequestQueue` |
| `GET` | `/api/presentations` | — | `200 [ Presentation... ]` (optional `?status=BUILDING\|REVIEWING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllPresentations`) | `PresentationEndpoint` ← `PresentationsView` |
| `GET` | `/api/presentations/{id}` | — | `200 Presentation` / `404` | `PresentationEndpoint` ← `PresentationsView` |
| `GET` | `/api/presentations/sse` | — | `text/event-stream` (one event per presentation change) | `PresentationEndpoint` ← `PresentationsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/presentations`

- Missing `targetAudience` → `"general"`.
- Missing `slideCount` → `5` (from configuration `brand-presentation.refinement.default-slide-count`).
- Missing `wordsPerSlide` → `80` (from `brand-presentation.refinement.default-words-per-slide`).
- Missing `requestedBy` → `"anonymous"`.
- `slideCount` must be in `[3, 20]`; otherwise `400`.
- `wordsPerSlide` must be in `[20, 300]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(topic, requestedBy)`; the second submission returns the first `presentationId` (200) instead of starting a new workflow.

## JSON shapes

### Presentation

```json
{
  "presentationId": "pr-8c4b1…",
  "topic": "launch of the new AI-assisted analytics dashboard",
  "targetAudience": "enterprise sales team",
  "slideCount": 5,
  "wordsPerSlide": 80,
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "slideSet": {
        "slides": [
          { "slideNumber": 1, "title": "Analytics Dashboard: A New Standard", "body": "Your sales pipeline deserves instant visibility. The AI-assisted analytics dashboard delivers real-time forecasts, automated territory reconciliation, and consistent data across every region — so your team closes faster with less effort.", "wordCount": 36 },
          { "slideNumber": 2, "title": "What We're Covering Today", "body": "Let's dive into the three problems you face, how the dashboard resolves each one, a real customer outcome, and your recommended next step.", "wordCount": 26 }
        ],
        "totalWordCount": 152,
        "builtAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "brandReview": {
        "verdict": "REVISE",
        "feedback": {
          "bullets": [
            "Slide 2 uses informal phrase 'Let's dive into' — replace with a formal transition.",
            "Slide 4 missing; deck is structurally incomplete for a 5-slide brief.",
            "Closing slide has no specific call-to-action with a measurable next step."
          ],
          "overallRationale": "Tone and CTA fall below threshold despite sound content on slides 1 and 3."
        },
        "score": 2,
        "reviewedAt": "2026-06-28T09:01:18Z"
      }
    },
    {
      "attemptNumber": 2,
      "slideSet": {
        "slides": [
          { "slideNumber": 1, "title": "Analytics Dashboard: A New Standard", "body": "Your sales pipeline deserves instant visibility. The AI-assisted analytics dashboard delivers real-time forecasts, automated territory reconciliation, and consistent data across every region — so your team closes faster with less effort.", "wordCount": 36 },
          { "slideNumber": 2, "title": "What We're Covering Today", "body": "The following slides present three challenges the enterprise sales team faces, the dashboard's targeted response to each, a documented customer outcome, and the recommended next step for your organisation.", "wordCount": 32 }
        ],
        "totalWordCount": 148,
        "builtAt": "2026-06-28T09:01:29Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "brandReview": {
        "verdict": "APPROVE",
        "feedback": {
          "bullets": [],
          "overallRationale": "Tone, hierarchy, CTA, and competitive neutrality all meet or exceed threshold."
        },
        "score": 5,
        "reviewedAt": "2026-06-28T09:01:44Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedSlideSet": { "slides": [ "…" ], "totalWordCount": 148, "builtAt": "2026-06-28T09:01:29Z" },
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:58Z",
  "finishedAt": "2026-06-28T09:01:45Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",             "detail": "" }
{ "passed": false, "reasonCode": "OVER_WORD_LIMIT", "detail": "Slides 3, 4 exceed the 80-word ceiling." }
```

`reasonCode` is one of `OK`, `OVER_WORD_LIMIT`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: presentation-update
data: { "presentationId": "pr-8c4b1…", "status": "REVIEWING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `presentationId`. The full `Presentation` JSON is included so a fresh client can render the row without a separate fetch.
