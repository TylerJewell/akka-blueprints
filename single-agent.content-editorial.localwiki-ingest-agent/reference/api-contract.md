# API contract — localwiki-ingest-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/ingests` | `SubmitIngestRequest` | `201 { ingestId }` | `IngestEndpoint` → `IngestEntity` |
| `GET` | `/api/ingests` | — | `200 [ Ingest... ]` (newest-first) | `IngestEndpoint` ← `WikiPageView` |
| `GET` | `/api/ingests/{id}` | — | `200 Ingest` / `404` | `IngestEndpoint` ← `WikiPageView` |
| `GET` | `/api/ingests/sse` | — | `text/event-stream` | `IngestEndpoint` ← `WikiPageView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitIngestRequest (request body)

```json
{
  "sourceType": "URL",
  "sourceAddress": "https://example.com/articles/intro-to-akka",
  "targetNamespace": "/docs",
  "submittedBy": "editor-7"
}
```

`sourceType` must be one of `URL`, `IMAGE`, `PDF`.

### Ingest (response body)

```json
{
  "ingestId": "ig-4fa...",
  "submission": {
    "ingestId": "ig-4fa...",
    "sourceType": "URL",
    "sourceAddress": "https://example.com/articles/intro-to-akka",
    "targetNamespace": "/docs",
    "submittedBy": "editor-7",
    "submittedAt": "2026-06-28T09:15:00Z"
  },
  "sanitized": {
    "redactedText": "Introduction to Akka\n\nAkka is a toolkit for building ... Contact: [REDACTED-EMAIL]",
    "mimeType": "text/plain",
    "piiCategoriesFound": ["email"]
  },
  "page": {
    "title": "Introduction to Akka",
    "slug": "introduction-to-akka",
    "summary": "An overview of Akka's actor model and distributed systems primitives.",
    "body": "## Overview\n\nAkka provides ...\n\n## Key Concepts\n\n...",
    "categoryTags": ["akka", "distributed-systems", "actors"],
    "targetPath": "/docs/introduction-to-akka",
    "filedAt": "2026-06-28T09:15:22Z"
  },
  "status": "PAGE_FILED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:22Z",
  "failureReason": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: ingest-update
data: { "ingestId": "ig-4fa...", "status": "PAGE_FILED", "page": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `INGESTING`, `PAGE_FILED`, `FAILED`). Clients reconcile by `ingestId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
