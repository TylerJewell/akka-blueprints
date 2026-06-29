# API contract — brand-aligner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/materials` | `{ "topic": String, "targetAudience"?: String, "wordCeiling"?: Integer, "requestedBy"?: String }` | `202 { "materialId": String }` | `BrandEndpoint` → `BriefQueue` |
| `GET` | `/api/materials` | — | `200 [ Material... ]` (optional `?status=DRAFTING\|REVIEWING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllMaterials`) | `BrandEndpoint` ← `MaterialsView` |
| `GET` | `/api/materials/{id}` | — | `200 Material` / `404` | `BrandEndpoint` ← `MaterialsView` |
| `GET` | `/api/materials/sse` | — | `text/event-stream` (one event per material change) | `BrandEndpoint` ← `MaterialsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/materials`

- Missing `wordCeiling` → 150 (from `brand-aligner.alignment.default-word-ceiling`).
- Missing `targetAudience` → `"general"`.
- Missing `requestedBy` → `"anonymous"`.
- `wordCeiling` must be in `[10, 1000]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(topic, requestedBy)`; the second submission returns the first `materialId` (200) instead of starting a new workflow.

## JSON shapes

### Material

```json
{
  "materialId": "m-9b1c4…",
  "topic": "Launch announcement for a new developer API that cuts integration time",
  "targetAudience": "backend engineers",
  "wordCeiling": 150,
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "variant": {
        "text": "Ship integrations in hours, not weeks. …",
        "wordCount": 142,
        "generatedAt": "2026-06-28T09:01:02Z"
      },
      "compliance": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Sentence 2 claims \"10× faster\" without qualification — add \"up to\" or cite a benchmark.",
            "Opening uses \"revolutionary\", an unsubstantiated superlative — replace with a specific benefit."
          ],
          "overallRationale": "Claim accuracy falls below threshold despite sound tone and audience fit."
        },
        "score": 3,
        "reviewedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "variant": {
        "text": "Ship integrations in hours, not weeks. …",
        "wordCount": 138,
        "generatedAt": "2026-06-28T09:01:21Z"
      },
      "compliance": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "APPROVE",
        "notes": { "bullets": [], "overallRationale": "Tone, audience fit, claim accuracy, and brand-safety all meet threshold." },
        "score": 5,
        "reviewedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedText": "Ship integrations in hours, not weeks. …",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Compliance verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "OVER_WORD_CEILING", "detail": "Variant is 178 words; ceiling is 150." }
```

`reasonCode` is one of `OK`, `OVER_WORD_CEILING`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: material-update
data: { "materialId": "m-9b1c4…", "status": "REVIEWING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `materialId`. The full `Material` JSON is included so a fresh client can render the row without a separate fetch.
