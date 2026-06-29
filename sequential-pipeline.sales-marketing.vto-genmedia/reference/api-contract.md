# API contract — vto-genmedia

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tryons` | `SubmitTryOnRequest` | `201 { tryOnId }` | `TryOnEndpoint` → `TryOnEntity` |
| `GET` | `/api/tryons` | — | `200 [ TryOnRecord... ]` (newest-first) | `TryOnEndpoint` ← `TryOnView` |
| `GET` | `/api/tryons/{id}` | — | `200 TryOnRecord` / `404` | `TryOnEndpoint` ← `TryOnView` |
| `GET` | `/api/tryons/sse` | — | `text/event-stream` | `TryOnEndpoint` ← `TryOnView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTryOnRequest (request body)

```json
{
  "garmentId": "summer-dress-blue",
  "modelPreset": "standing-front",
  "includeVideo": true
}
```

### TryOnRecord (response body)

```json
{
  "tryOnId": "t-8bc7d1...",
  "garmentId": "summer-dress-blue",
  "modelPreset": "standing-front",
  "includeVideo": true,
  "assets": {
    "garment": {
      "garmentId": "summer-dress-blue",
      "displayName": "Summer Dress — Ocean Blue",
      "category": "dress",
      "primaryColour": "#1E90FF",
      "imageUrl": "https://cdn.example.com/garments/summer-dress-blue.png"
    },
    "preset": {
      "presetId": "standing-front",
      "pose": "standing-front",
      "backgroundColour": "#F5F5F5",
      "targetWidthPx": 800,
      "targetHeightPx": 1200
    },
    "includeVideo": true,
    "preparedAt": "2026-06-28T10:00:00Z"
  },
  "media": {
    "composite": {
      "assetId": "img-7a3b",
      "url": "https://cdn.example.com/generated/summer-dress-blue-standing-front.png",
      "widthPx": 800,
      "heightPx": 1200,
      "mimeType": "image/png"
    },
    "videoClip": {
      "assetId": "vid-2c9d",
      "url": "https://cdn.example.com/generated/summer-dress-blue-standing-front.mp4",
      "durationMs": 3000,
      "mimeType": "video/mp4"
    },
    "generationParams": {
      "model": "genmedia-v2",
      "backgroundColour": "#F5F5F5"
    },
    "generatedAt": "2026-06-28T10:00:20Z"
  },
  "validatedMedia": {
    "composite": {
      "assetId": "img-7a3b",
      "url": "https://cdn.example.com/generated/summer-dress-blue-standing-front.png",
      "widthPx": 800,
      "heightPx": 1200,
      "mimeType": "image/png"
    },
    "videoClip": {
      "assetId": "vid-2c9d",
      "url": "https://cdn.example.com/generated/summer-dress-blue-standing-front.mp4",
      "durationMs": 3000,
      "mimeType": "video/mp4"
    },
    "dimensionReport": {
      "aspectRatioOk": true,
      "expected": "2:3",
      "actual": "2:3"
    },
    "colourReport": {
      "colourFidelityOk": true,
      "deltaE": 4.2,
      "dominantActual": "#1A8FFF",
      "expectedPrimary": "#1E90FF"
    },
    "safetyVerdict": "CLEARED",
    "validatedAt": "2026-06-28T10:00:40Z"
  },
  "quality": {
    "score": 5,
    "rationale": "Aspect ratio, colour fidelity, composite presence, and video completeness all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:41Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:41Z",
  "safetyRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `safetyRejections` is an array — empty on the happy path, populated when the guardrail intercepted an unsafe output.

A record where the safety guardrail fired on one iteration:

```json
{
  "tryOnId": "t-3ef2a0...",
  "status": "EVALUATED",
  "safetyRejections": [
    {
      "category": "EXPLICIT",
      "reason": "safety-violation: EXPLICIT — composite url contains 'unsafe-fixture'",
      "rejectedAt": "2026-06-28T10:01:05Z"
    }
  ]
}
```

### SSE event format

```
event: tryon-update
data: { "tryOnId": "t-8bc7d1...", "status": "MEDIA_GENERATED", "media": { ... }, ... }
```

One event per state transition (`CREATED`, `PREPARING`, `ASSETS_READY`, `GENERATING`, `MEDIA_GENERATED`, `VALIDATING`, `VALIDATED`, `EVALUATED`, `FAILED`) and one per `SafetyRejected` audit event:

```
event: tryon-safety-rejection
data: { "tryOnId": "t-8bc7d1...", "category": "EXPLICIT", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `tryOnId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitTryOnRequest` record and the `TryOnCreated` event to carry it.
