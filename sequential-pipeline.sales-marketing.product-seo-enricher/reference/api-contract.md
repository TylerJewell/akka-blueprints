# API contract — product-seo-enricher

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/enrichments` | `SubmitEnrichmentRequest` | `201 { enrichmentId }` | `EnrichmentEndpoint` → `EnrichmentEntity` |
| `GET` | `/api/enrichments` | — | `200 [ EnrichmentRecord... ]` (newest-first) | `EnrichmentEndpoint` ← `EnrichmentView` |
| `GET` | `/api/enrichments/{id}` | — | `200 EnrichmentRecord` / `404` | `EnrichmentEndpoint` ← `EnrichmentView` |
| `GET` | `/api/enrichments/sse` | — | `text/event-stream` | `EnrichmentEndpoint` ← `EnrichmentView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitEnrichmentRequest (request body)

```json
{
  "productName": "UltraFit Running Shoe X9"
}
```

### EnrichmentRecord (response body)

```json
{
  "enrichmentId": "e-3bc94a...",
  "productName": "UltraFit Running Shoe X9",
  "serpResult": {
    "entries": [
      {
        "title": "Best trail running shoes 2026 — UltraFit X9 review",
        "url": "https://example.org/runnerworld/ultrafit-x9-review",
        "position": 1,
        "snippet": "The UltraFit X9 delivers superior cushioning on technical terrain.",
        "fetchedAt": "2026-06-28T10:00:00Z"
      }
    ],
    "fetchedAt": "2026-06-28T10:00:00Z"
  },
  "serpAnalysis": {
    "keywords": [
      { "keyword": "trail running shoe", "frequency": 2, "sourceUrl": "https://example.org/runnerworld/ultrafit-x9-review" }
    ],
    "competitors": [
      { "domain": "example.org", "title": "UltraFit X9 vs TrailMax Pro — 2026 comparison", "position": 2, "angle": "direct head-to-head with TrailMax Pro" }
    ],
    "analyzedAt": "2026-06-28T10:00:05Z"
  },
  "enrichment": {
    "productName": "UltraFit Running Shoe X9",
    "rankedKeywords": [
      { "keyword": "trail running shoe", "relevanceScore": 1.0, "rationale": "appears in 2 of 2 SERP entries" }
    ],
    "competitorSummary": "example.org positions a direct comparison with TrailMax Pro at position 2.",
    "recommendedMetaDescription": "UltraFit Running Shoe X9 — premium trail running shoe with superior cushioning for long-distance terrain.",
    "writtenAt": "2026-06-28T10:00:10Z"
  },
  "quality": {
    "score": 5,
    "rationale": "Keyword coverage, meta-description length, competitor grounding, and keyword-list non-emptiness all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:10Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:10Z",
  "guardrailRejections": []
}
```

A rejection example (populated when `BrowserGuardrail` caught a misordered tool call):

```json
{
  "guardrailRejections": [
    {
      "phase": "FETCH",
      "tool": "rankKeywords",
      "reason": "phase-violation: rankKeywords requires status in {ANALYZED, ENRICHING} with serpAnalysis present, saw FETCHING",
      "rejectedAt": "2026-06-28T10:00:01Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path.

### SSE event format

```
event: enrichment-update
data: { "enrichmentId": "e-3bc94a...", "status": "ANALYZED", "serpAnalysis": { ... }, ... }
```

One event per state transition (`CREATED`, `FETCHING`, `FETCHED`, `ANALYZING`, `ANALYZED`, `ENRICHING`, `ENRICHED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: enrichment-rejection
data: { "enrichmentId": "e-3bc94a...", "phase": "FETCH", "tool": "rankKeywords", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `enrichmentId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitEnrichmentRequest` record and the `EnrichmentCreated` event to carry it.
