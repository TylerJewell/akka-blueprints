# API contract — ad-creator-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/ad-jobs` | `SubmitAdJobRequest` | `201 { adJobId }` | `AdJobEndpoint` → `AdJobEntity` |
| `GET` | `/api/ad-jobs` | — | `200 [ AdJobRecord... ]` (newest-first) | `AdJobEndpoint` ← `AdJobView` |
| `GET` | `/api/ad-jobs/{id}` | — | `200 AdJobRecord` / `404` | `AdJobEndpoint` ← `AdJobView` |
| `GET` | `/api/ad-jobs/sse` | — | `text/event-stream` | `AdJobEndpoint` ← `AdJobView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAdJobRequest (request body)

```json
{
  "productUrl": "https://example.com/products/zenith-wireless"
}
```

### AdJobRecord (response body)

```json
{
  "adJobId": "aj-7c2d1f...",
  "productUrl": "https://example.com/products/zenith-wireless",
  "profile": {
    "productName": "Zenith Wireless Headphones",
    "productUrl": "https://example.com/products/zenith-wireless",
    "attributes": [
      { "name": "battery life", "value": "40 hours per charge" },
      { "name": "noise cancellation", "value": "active noise cancellation with three adjustable levels" }
    ],
    "targetAudience": "commuters and remote workers",
    "brandConstraints": ["no competitor comparisons", "focus on comfort and endurance"],
    "scrapedAt": "2026-06-28T10:00:00Z"
  },
  "copy": {
    "variants": [
      {
        "format": "INSTAGRAM",
        "headline": "40 hours. Zero interruptions.",
        "body": "Zenith Wireless Headphones deliver 40 hours of playtime per charge with three-level active noise cancellation — built for commuters who can't afford a dead battery.",
        "callToAction": "Shop now"
      },
      {
        "format": "FACEBOOK",
        "headline": "Work anywhere. Hear nothing you don't want to.",
        "body": "With 40-hour battery life and adjustable active noise cancellation, Zenith Wireless Headphones are built for long days away from a desk.",
        "callToAction": "Learn more"
      },
      {
        "format": "SEARCH",
        "headline": "Zenith Wireless – 40hr Battery, Active Noise Cancellation",
        "body": "Designed for commuters. 40-hour battery, three-level ANC, premium sound.",
        "callToAction": "Buy now"
      }
    ],
    "primaryHeadline": "40 hours. Zero interruptions.",
    "primaryBody": "Zenith Wireless Headphones deliver 40 hours of playtime per charge with three-level active noise cancellation — built for commuters who can't afford a dead battery.",
    "callToAction": "Shop now",
    "draftedAt": "2026-06-28T10:00:05Z"
  },
  "visual": {
    "imagePrompt": "Studio product photograph of Zenith Wireless Headphones on a minimalist white surface, emphasising the premium matte-black finish and ear-cushion detail; soft directional lighting, 1:1 aspect ratio, clean background",
    "aspectRatio": "1:1",
    "styleGuidance": "Clean, premium, minimal props; no human models",
    "generatedAt": "2026-06-28T10:00:10Z"
  },
  "adPackage": {
    "title": "Zenith Wireless Headphones — Ad Package",
    "copy": { "...": "see copy above" },
    "visual": { "...": "see visual above" },
    "assembledAt": "2026-06-28T10:00:10Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Attribute grounding, call-to-action presence, visual-copy alignment, and variant coverage all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:11Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:11Z",
  "scrapingRejections": [],
  "brandSafetyRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `scrapingRejections` and `brandSafetyRejections` are arrays — empty on the happy path, populated when their respective guardrails fired.

### SSE event format

```
event: ad-job-update
data: { "adJobId": "aj-7c2d1f...", "status": "COPYING", "profile": { ... }, ... }
```

One event per state transition (`CREATED`, `SCRAPING`, `SCRAPED`, `COPYING`, `COPIED`, `GENERATING_VISUAL`, `VISUAL_GENERATED`, `EVALUATED`, `FAILED`) and one per audit rejection event:

```
event: scraping-rejection
data: { "adJobId": "aj-7c2d1f...", "url": "https://competitor.example.com/...", "reason": "scraping-policy-violation: url not on allow-list", "rejectedAt": "..." }

event: brand-safety-rejection
data: { "adJobId": "aj-7c2d1f...", "violations": [{ "rule": "disallowed_superlative", "offendingText": "best headphones guaranteed", "suggestion": "Remove superlative; use specific attribute instead." }], "rejectedAt": "..." }
```

Clients reconcile by `adJobId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitAdJobRequest` record and the `AdJobCreated` event to carry it.
