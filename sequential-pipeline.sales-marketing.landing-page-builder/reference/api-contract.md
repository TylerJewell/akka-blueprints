# API contract — landing-page-builder

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/generate` | `{ "concept": "string" }` | `{ "pageId": "uuid" }` | GenerationEndpoint |
| GET | `/api/pages?status=...` | — | `{ "pages": [Page, ...] }` | GenerationEndpoint |
| GET | `/api/pages/{pageId}` | — | `Page` or 404 | GenerationEndpoint |
| GET | `/api/pages/sse` | — | SSE stream of `Page` | GenerationEndpoint |
| GET | `/api/scrape?url=...` | — | template markup (200) or 403 | ScraperEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | GenerationEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | GenerationEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | GenerationEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static resource | AppEndpoint |

The `status` query param on `/api/pages` is filtered client-side in the endpoint
from the unfiltered `getAllPages` view query (Akka cannot auto-index the enum
column — Lesson 2).

## Page JSON form

Lifecycle fields are nullable — `Optional<T>` in Java, serialized as raw value or
`null` on the wire (Lesson 6).

```json
{
  "id": "uuid",
  "concept": "string or null",
  "status": "QUEUED | ANALYZED | SELECTED | CUSTOMIZED | PUBLISHED | REJECTED",
  "analyzedAt": "ISO-8601 or null",
  "brief": "string or null",
  "selectedAt": "ISO-8601 or null",
  "templateName": "string or null",
  "templateUrl": "string or null",
  "customizedAt": "ISO-8601 or null",
  "html": "string or null",
  "validatedAt": "ISO-8601 or null",
  "lintPassed": "boolean or null",
  "publishedAt": "ISO-8601 or null",
  "publishedUrl": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectReason": "string or null"
}
```

## Scrape response

`/api/scrape?url=` returns the canned markup mapped to an allowlisted URL with
`200 text/html`. Any off-allowlist or robots-disallowed URL returns `403` with
`{ "blocked": true, "reason": "string" }`.

## SSE event format

`/api/pages/sse` emits one event per `Page` update:

```
event: page
data: { ...Page JSON as above... }
```

Clients render the full list from the stream; each event replaces the matching
`id` in the client's local map.
