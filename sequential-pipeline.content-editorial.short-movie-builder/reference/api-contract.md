# API contract — short-movie-agents

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/productions` | `SubmitProductionRequest` | `201 { productionId }` | `MovieEndpoint` → `MovieEntity` |
| `GET` | `/api/productions` | — | `200 [ MovieRecord... ]` (newest-first) | `MovieEndpoint` ← `MovieView` |
| `GET` | `/api/productions/{id}` | — | `200 MovieRecord` / `404` | `MovieEndpoint` ← `MovieView` |
| `GET` | `/api/productions/sse` | — | `text/event-stream` | `MovieEndpoint` ← `MovieView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitProductionRequest (request body)

```json
{
  "brief": "a 60-second workplace-safety training video"
}
```

### MovieRecord (response body)

```json
{
  "productionId": "p-3bc7e1...",
  "brief": "a 60-second workplace-safety training video",
  "script": {
    "scenes": [
      {
        "sceneId": "intro-hazard",
        "description": "Worker approaches unmarked chemical spill.",
        "dialogueLine": "Always check the aisle before entering.",
        "type": "INTRO"
      }
    ],
    "genre": "training",
    "writtenAt": "2026-06-28T10:00:00Z"
  },
  "storyboard": {
    "shots": [
      { "shotId": "s-c4d5e6f7", "sceneId": "intro-hazard", "framing": "wide-shot", "durationSeconds": 6 }
    ],
    "designedAt": "2026-06-28T10:00:06Z"
  },
  "assembledPackage": {
    "packageScenes": [
      {
        "packageSceneId": "ps-001",
        "sceneId": "intro-hazard",
        "shotId": "s-c4d5e6f7",
        "assetPlaceholder": "asset-s-c4d5e6f7-intro-hazard",
        "orderIndex": 0
      }
    ],
    "totalRuntimeSeconds": 6,
    "assembledAt": "2026-06-28T10:00:12Z"
  },
  "reviewResult": {
    "coherenceChecks": [
      { "sceneId": "intro-hazard", "coherent": true, "note": "shot sceneId matches" }
    ],
    "summary": "All scenes coherent.",
    "reviewedAt": "2026-06-28T10:00:18Z"
  },
  "coherenceResult": {
    "score": 5,
    "rationale": "Scene coverage, shot reference validity, runtime plausibility, and order consistency all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:18Z"
  },
  "status": "REVIEWED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:18Z",
  "safetyBlocks": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `safetyBlocks` is an array — empty on the happy path, populated when a phase result was rejected by `ContentSafetyGuard`.

**Safety-blocked production (partial response):**

```json
{
  "productionId": "p-9fa2c0...",
  "brief": "a product teaser for a fictional smartwatch",
  "script": null,
  "status": "SCRIPTING",
  "safetyBlocks": [
    {
      "phase": "SCRIPT",
      "violationType": "regulated-brand",
      "reason": "safety-block: regulated-brand: script text references 'AppleWatch' as a fictional entity",
      "blockedAt": "2026-06-28T10:00:03Z"
    }
  ]
}
```

### SSE event format

```
event: production-update
data: { "productionId": "p-3bc7e1...", "status": "ASSEMBLED", "assembledPackage": { ... }, ... }
```

One event per state transition (`CREATED`, `SCRIPTING`, `SCRIPTED`, `STORYBOARDING`, `STORYBOARDED`, `ASSEMBLING`, `ASSEMBLED`, `REVIEWING`, `REVIEWED`, `FAILED`) and one per `SafetyBlockRecorded` audit event:

```
event: production-safety-block
data: { "productionId": "p-3bc7e1...", "phase": "SCRIPT", "violationType": "regulated-brand", "reason": "...", "blockedAt": "..." }
```

Clients reconcile by `productionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend `SubmitProductionRequest` and `ProductionCreated` to carry it.
