# API contract — content-pipeline

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/topics` | `{ "topic": "string" }` | `{ "articleId": "uuid" }` | ContentEndpoint → InboundTopicQueue |
| GET | `/api/articles?status=...` | — | `{ "articles": [Article, ...] }` | ContentEndpoint ← ArticlesView |
| GET | `/api/articles/{articleId}` | — | `Article` or 404 | ContentEndpoint ← ArticlesView |
| GET | `/api/articles/sse` | — | SSE stream of `Article` | ContentEndpoint ← ArticlesView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ContentEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ContentEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ContentEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

`?status=` filtering is applied client-side in the endpoint after `getAllArticles()` — the view cannot auto-index the enum column (Lesson 2).

## Payload shapes

`POST /api/topics` request:

```json
{ "topic": "How event sourcing changes audit trails" }
```

`POST /api/topics` response:

```json
{ "articleId": "5f2c…" }
```

`Article` (lifecycle fields are nullable — `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "topic": "string or null",
  "status": "RESEARCHING | WRITING | CRITIQUING | PUBLISHED | FAILED",
  "researchedAt": "ISO-8601 or null",
  "researchSummary": "string or null",
  "researchSources": "string or null",
  "draftedAt": "ISO-8601 or null",
  "title": "string or null",
  "body": "string or null",
  "critiquedAt": "ISO-8601 or null",
  "critiqueScore": "number or null",
  "critiqueNotes": "string or null",
  "publishedAt": "ISO-8601 or null",
  "publishedUrl": "string or null",
  "failedAt": "ISO-8601 or null",
  "failureReason": "string or null"
}
```

## SSE event format

`GET /api/articles/sse` streams the view via `componentClient.forView().method(ArticlesView::streamAllArticles).source()` mapped to Server-Sent Events. Each event:

```
event: article
data: { …Article JSON as above… }
```

The browser subscribes with `EventSource` and upserts each article into the live list by `id`. No polling.
