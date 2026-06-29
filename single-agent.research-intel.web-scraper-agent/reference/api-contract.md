# API contract — web-scraper-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/scrapes` | `SubmitScrapeRequest` | `201 { scrapeId }` | `ScrapeEndpoint` → `ScrapeEntity` |
| `GET` | `/api/scrapes` | — | `200 [ Scrape... ]` (newest-first) | `ScrapeEndpoint` ← `ScrapeView` |
| `GET` | `/api/scrapes/{id}` | — | `200 Scrape` / `404` | `ScrapeEndpoint` ← `ScrapeView` |
| `GET` | `/api/scrapes/sse` | — | `text/event-stream` | `ScrapeEndpoint` ← `ScrapeView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitScrapeRequest (request body)

```json
{
  "targetUrl": "https://news.example.net/transit-plan-approved",
  "extractionSchema": "news-article",
  "submittedBy": "analyst-42"
}
```

### Scrape (response body)

```json
{
  "scrapeId": "s-7a3c...",
  "request": {
    "scrapeId": "s-7a3c...",
    "targetUrl": "https://news.example.net/transit-plan-approved",
    "extractionSchema": "news-article",
    "submittedBy": "analyst-42",
    "submittedAt": "2026-06-28T14:22:00Z"
  },
  "result": {
    "title": "City Council Approves New Transit Plan",
    "summary": "The city council voted 7-2 to approve a $340 million transit expansion plan.",
    "dataPoints": [
      {
        "fieldName": "author",
        "value": "[REDACTED-PERSON-NAME]",
        "sourceQuote": "By [REDACTED-PERSON-NAME], Staff Reporter"
      },
      {
        "fieldName": "publishDate",
        "value": "2026-06-15",
        "sourceQuote": "Published June 15, 2026"
      }
    ],
    "fetchedUrl": "https://news.example.net/transit-plan-approved",
    "httpStatus": 200,
    "extractedAt": "2026-06-28T14:22:18Z"
  },
  "sanitized": {
    "sanitizedSummary": "The city council voted 7-2 to approve a $340 million transit expansion plan.",
    "sanitizedDataPoints": [
      {
        "fieldName": "author",
        "value": "[REDACTED-PERSON-NAME]",
        "sourceQuote": "By [REDACTED-PERSON-NAME], Staff Reporter"
      }
    ],
    "piiCategoriesFound": ["person-name", "email"]
  },
  "status": "SANITIZED",
  "blockedReason": null,
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

A blocked scrape response looks like:

```json
{
  "scrapeId": "s-9b1d...",
  "request": { ... },
  "result": null,
  "sanitized": null,
  "status": "BLOCKED",
  "blockedReason": "domain-not-allowed: restricted.internal.example.com",
  "createdAt": "2026-06-28T14:25:00Z",
  "finishedAt": "2026-06-28T14:25:01Z"
}
```

### SSE event format

```
event: scrape-update
data: { "scrapeId": "s-7a3c...", "status": "SANITIZED", "result": { ... }, "sanitized": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `FETCHING`, `EXTRACTED`, `SANITIZED`, `BLOCKED`, `FAILED`). Clients reconcile by `scrapeId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body. Domain allowlist management (adding/removing entries from `allowed-domains.jsonl`) is a deployment-time configuration change, not an API operation in this baseline.
