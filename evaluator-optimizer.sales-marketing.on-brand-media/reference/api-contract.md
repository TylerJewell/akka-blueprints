# API contract — on-brand-genmedia

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/assets` | `{ "product": String, "channel": String, "tone"?: String, "tokenCeiling"?: Integer, "requestedBy"?: String }` | `202 { "assetId": String }` | `BrandEndpoint` → `CampaignQueue` |
| `GET` | `/api/assets` | — | `200 [ Asset... ]` (optional `?status=GENERATING\|REVIEWING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllAssets`) | `BrandEndpoint` ← `AssetsView` |
| `GET` | `/api/assets/{id}` | — | `200 Asset` / `404` | `BrandEndpoint` ← `AssetsView` |
| `GET` | `/api/assets/sse` | — | `text/event-stream` (one event per asset change) | `BrandEndpoint` ← `AssetsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/assets`

- Missing `tone` → `"professional"`.
- Missing `tokenCeiling` → `300` (from `on-brand-genmedia.generation.default-token-ceiling`).
- Missing `requestedBy` → `"anonymous"`.
- `tokenCeiling` must be in `[50, 2000]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(product, channel, requestedBy)`; the second submission returns the first `assetId` (200) instead of starting a new workflow.

## JSON shapes

### Asset

```json
{
  "assetId": "a-9c3d1…",
  "product": "Nova Project Management Platform",
  "channel": "LinkedIn",
  "tone": "professional",
  "tokenCeiling": 300,
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "asset": {
        "headline": "Nova brings your project timeline under one roof",
        "bodyCopy": "Nova centralises task assignment, milestone tracking, and budget forecasting in a single workspace.",
        "socialCaption": "Distributed teams trust Nova for every project milestone. #ProjectManagement",
        "tokenCount": 285,
        "generatedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Body copy states '30% fewer meetings' without qualification — remove or attribute.",
            "Social caption is 52 words, exceeding the LinkedIn 40-word recommendation.",
            "Headline passive construction weakens the professional register."
          ],
          "overallRationale": "Product accuracy and channel fit fall below threshold."
        },
        "score": 2,
        "reviewedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "asset": {
        "headline": "Nova puts every project milestone in one workspace",
        "bodyCopy": "Nova delivers task assignment with dependency mapping, milestone alerts, and live budget tracking. Each feature is available at launch.",
        "socialCaption": "Task dependencies, milestone alerts, and live budget — all in one place with Nova. #ProjectManagement",
        "tokenCount": 271,
        "generatedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "APPROVE",
        "notes": { "bullets": [], "overallRationale": "Voice consistent; capabilities named specifically; LinkedIn format appropriate." },
        "score": 5,
        "reviewedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedAsset": {
    "headline": "Nova puts every project milestone in one workspace",
    "bodyCopy": "Nova delivers task assignment with dependency mapping, milestone alerts, and live budget tracking. Each feature is available at launch.",
    "socialCaption": "Task dependencies, milestone alerts, and live budget — all in one place with Nova. #ProjectManagement",
    "tokenCount": 271,
    "generatedAt": "2026-06-28T09:01:21Z"
  },
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",                "detail": "" }
{ "passed": false, "reasonCode": "PROHIBITED_CONTENT", "detail": "Asset contains prohibited term 'world's best' in headline." }
```

`reasonCode` is one of `OK`, `PROHIBITED_CONTENT`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: asset-update
data: { "assetId": "a-9c3d1…", "status": "REVIEWING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `assetId`. The full `Asset` JSON is included so a fresh client can render the row without a separate fetch.
