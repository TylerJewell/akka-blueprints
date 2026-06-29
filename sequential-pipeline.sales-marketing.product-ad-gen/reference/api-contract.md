# API contract — product-catalog-ad-generation

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/ad-jobs` | `SubmitAdJobRequest` | `201 { jobId }` | `AdJobEndpoint` → `AdJobEntity` |
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
  "productName": "Wireless Noise-Cancelling Headphones"
}
```

### AdJobRecord (response body)

```json
{
  "jobId": "j-3bc7d1...",
  "productName": "Wireless Noise-Cancelling Headphones",
  "enrichedProduct": {
    "productId": "prod-wnch-001",
    "productName": "Wireless Noise-Cancelling Headphones",
    "attributes": {
      "category": "consumer-electronics",
      "pricingTier": "premium",
      "keyFeatures": ["adaptive ANC", "40-hour battery", "foldable design"],
      "targetAudience": "remote workers and frequent travellers"
    },
    "enrichedAt": "2026-06-28T10:00:02Z"
  },
  "adDraft": {
    "headline": "ClearSound Pro — Silence Everything Else",
    "callToAction": "shop-clearsound-pro",
    "placements": [
      { "type": "search", "copy": "ClearSound Pro wireless headphones. 40hr battery, ANC, foldable. Free shipping over $50.", "charCount": 87 },
      { "type": "display", "copy": "Immerse yourself in pure audio. ClearSound Pro's adaptive noise cancellation reads your environment and adjusts in real time.", "charCount": 130 },
      { "type": "social", "copy": "New drop: ClearSound Pro 40hr battery. Zero distractions. Built for focused minds. #ClearSound", "charCount": 93 }
    ],
    "draftedAt": "2026-06-28T10:00:08Z"
  },
  "productAd": {
    "jobId": "j-3bc7d1...",
    "headline": "ClearSound Pro — Silence Everything Else",
    "callToAction": "shop-clearsound-pro",
    "placements": [
      { "type": "search", "copy": "ClearSound Pro wireless headphones. 40hr battery, ANC, foldable. Free shipping over $50.", "charCount": 87 },
      { "type": "display", "copy": "Immerse yourself in pure audio. ClearSound Pro's adaptive noise cancellation reads your environment and adjusts in real time.", "charCount": 130 },
      { "type": "social", "copy": "New drop: ClearSound Pro 40hr battery. Zero distractions. Built for focused minds. #ClearSound", "charCount": 93 }
    ],
    "approvedAt": "2026-06-28T10:00:14Z"
  },
  "complianceResult": {
    "score": 5,
    "rationale": "Headline element present, no prohibited words, CTA slug valid, placement parity satisfied.",
    "scoredAt": "2026-06-28T10:00:14Z"
  },
  "status": "SCORED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:14Z",
  "guardrailRejections": [
    {
      "phase": "DRAFT",
      "field": "headline",
      "rule": "prohibited-word",
      "found": "cheapest",
      "rejectedAt": "2026-06-28T10:00:06Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when the agent's draft or review response violated a brand-policy rule.

### SSE event format

```
event: ad-job-update
data: { "jobId": "j-3bc7d1...", "status": "DRAFTED", "adDraft": { ... }, ... }
```

One event per state transition (`CREATED`, `ENRICHING`, `ENRICHED`, `DRAFTING`, `DRAFTED`, `REVIEWING`, `APPROVED`, `SCORED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: ad-job-rejection
data: { "jobId": "j-3bc7d1...", "phase": "DRAFT", "field": "headline", "rule": "prohibited-word", "found": "cheapest", "rejectedAt": "..." }
```

Clients reconcile by `jobId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitAdJobRequest` record and the `AdJobCreated` event to carry it.
