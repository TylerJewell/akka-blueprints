# API contract — image-policy-scorer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/images` | `{ "promptText": String, "audienceTier"?: String, "requestedBy"?: String }` | `202 { "imageId": String }` | `ImageEndpoint` → `PromptQueue` |
| `GET` | `/api/images` | — | `200 [ Image... ]` (optional `?status=GENERATING\|SCORING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllImages`) | `ImageEndpoint` ← `ImagesView` |
| `GET` | `/api/images/{id}` | — | `200 Image` / `404` | `ImageEndpoint` ← `ImagesView` |
| `GET` | `/api/images/sse` | — | `text/event-stream` (one event per image change) | `ImageEndpoint` ← `ImagesView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/images`

- Missing `audienceTier` → `"general"`.
- Missing `requestedBy` → `"anonymous"`.
- `audienceTier` must be one of `general`, `mature`, `children`; otherwise `400`.
- `promptText` must be non-empty and at most 500 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(promptText, requestedBy)`; the second submission returns the first `imageId` (200) instead of starting a new workflow.

## JSON shapes

### Image

```json
{
  "imageId": "img-3a9f1…",
  "promptText": "a street food market at dusk in a busy city",
  "audienceTier": "general",
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "description": {
        "descriptionText": "Vendors in bright aprons serve steaming noodles from illuminated stalls along a cobblestone lane; amber twilight fills the sky behind the crowd.",
        "contentCategory": "lifestyle",
        "brandSafetySignal": "mild",
        "generatedAt": "2026-06-28T09:01:02Z"
      },
      "gateVerdict": { "passed": true, "reasonCode": "OK", "detail": "" },
      "policyVerdict": {
        "decision": "FAIL",
        "notes": {
          "bullets": [
            "Brand-safety signal 'mild' is ambiguous for the general tier; clarify whether alcohol signage is present.",
            "Crowd density and nightlife setting may conflict with all-ages brand guidelines.",
            "Prompt fidelity acceptable but imagery skews later in the evening than requested."
          ],
          "overallRationale": "Brand-safety and appropriateness dimensions fall below threshold."
        },
        "score": 3,
        "scoredAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "description": {
        "descriptionText": "Vendors in bright aprons serve steaming noodles and dumplings from illuminated wooden stalls along a cobblestone lane as the sky fades from amber to violet; stall signs display only food names and prices.",
        "contentCategory": "lifestyle",
        "brandSafetySignal": "none",
        "generatedAt": "2026-06-28T09:01:21Z"
      },
      "gateVerdict": { "passed": true, "reasonCode": "OK", "detail": "" },
      "policyVerdict": {
        "decision": "PASS",
        "notes": {
          "bullets": [],
          "overallRationale": "All four dimensions meet threshold; brand-safe, audience-appropriate, on-brief."
        },
        "score": 5,
        "scoredAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedDescription": "Vendors in bright aprons serve steaming noodles and dumplings from illuminated wooden stalls along a cobblestone lane as the sky fades from amber to violet; stall signs display only food names and prices.",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Safety gate verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "PROHIBITED_SIGNAL", "detail": "brandSafetySignal 'violence' is on the prohibited list." }
```

`reasonCode` is one of `OK`, `PROHIBITED_SIGNAL`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: image-update
data: { "imageId": "img-3a9f1…", "status": "SCORING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `imageId`. The full `Image` JSON is included so a fresh client can render the row without a separate fetch.
