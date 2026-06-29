# API contract — competitor-research-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/competitors` | `SubmitCompetitorRequest` | `201 { competitorId }` | `CompetitorEndpoint` → `CompetitorEntity` |
| `GET` | `/api/competitors` | — | `200 [ CompetitorRecord... ]` (newest-first) | `CompetitorEndpoint` ← `CompetitorView` |
| `GET` | `/api/competitors/{id}` | — | `200 CompetitorRecord` / `404` | `CompetitorEndpoint` ← `CompetitorView` |
| `GET` | `/api/competitors/sse` | — | `text/event-stream` | `CompetitorEndpoint` ← `CompetitorView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitCompetitorRequest (request body)

```json
{
  "name": "Acme Analytics"
}
```

### CompetitorRecord (response body)

```json
{
  "competitorId": "c-7bf3a1...",
  "name": "Acme Analytics",
  "searchResults": {
    "results": [
      {
        "title": "Acme Analytics — Product Overview",
        "url": "https://example.org/acme-analytics/product",
        "excerpt": "Acme Analytics offers a self-serve BI platform...",
        "fetchedAt": "2026-06-28T10:00:00Z"
      }
    ],
    "searchedAt": "2026-06-28T10:00:03Z"
  },
  "summary": {
    "fields": [
      { "fieldName": "pricing_model", "value": "Per-seat SaaS, $25/seat/month.", "supportingUrl": "https://example.org/acme-analytics/product" },
      { "fieldName": "primary_use_case", "value": "Self-serve BI for mid-market SaaS.", "supportingUrl": "https://example.org/acme-analytics/product" },
      { "fieldName": "notable_integrations", "value": "(not found)", "supportingUrl": "https://example.org/acme-analytics/product" },
      { "fieldName": "known_differentiators", "value": "Customer-cloud data residency.", "supportingUrl": "https://example.org/acme-analytics/series-b" },
      { "fieldName": "data_residency_stance", "value": "Customer-cloud by design.", "supportingUrl": "https://example.org/acme-analytics/series-b" }
    ],
    "domainClassification": "analytics",
    "summarizedAt": "2026-06-28T10:00:10Z"
  },
  "profile": {
    "competitorId": "c-7bf3a1...",
    "name": "Acme Analytics",
    "oneLiner": "Mid-market self-serve BI platform with customer-cloud data residency and per-seat pricing.",
    "fields": [
      { "fieldName": "pricing_model", "value": "Per-seat SaaS, $25/seat/month.", "supportingUrl": "https://example.org/acme-analytics/product" }
    ],
    "notionRef": {
      "pageId": "notion-page-abc123",
      "pageUrl": "https://notion.so/acme-analytics-abc123",
      "publishedAt": "2026-06-28T10:00:15Z"
    },
    "profiledAt": "2026-06-28T10:00:15Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Field coverage, source attribution, source provenance, and Notion ref all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:16Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:16Z",
  "guardrailRejections": []
}
```

A `guardrailRejections` entry looks like:

```json
{
  "phase": "SEARCH",
  "tool": "writeToNotion",
  "reason": "phase-violation: writeToNotion requires status in {SUMMARIZED, PUBLISHING} with summary present, saw SEARCHING",
  "rejectedAt": "2026-06-28T10:00:01Z"
}
```

Or for a schema violation:

```json
{
  "phase": "PUBLISH",
  "tool": "writeToNotion",
  "reason": "schema-violation: required field 'Pricing Model' is missing from the Notion page payload",
  "rejectedAt": "2026-06-28T10:00:14Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the guardrail caught a violation.

### SSE event format

```
event: competitor-update
data: { "competitorId": "c-7bf3a1...", "status": "SUMMARIZED", "summary": { ... }, ... }
```

One event per state transition (`CREATED`, `SEARCHING`, `SEARCHED`, `SUMMARIZING`, `SUMMARIZED`, `PUBLISHING`, `PUBLISHED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: competitor-rejection
data: { "competitorId": "c-7bf3a1...", "phase": "SEARCH", "tool": "writeToNotion", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `competitorId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitCompetitorRequest` record and the `CompetitorCreated` event to carry it.
