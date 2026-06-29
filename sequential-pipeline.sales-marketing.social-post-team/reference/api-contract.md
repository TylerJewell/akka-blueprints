# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Timestamps are ISO-8601 strings; lifecycle fields are nullable (`Optional<T>` in Java, `null` on the wire).

## Pipeline endpoints (PostEndpoint, `/api`)

### POST /api/concepts
Submit a creative concept. Enqueues onto `ConceptQueue`, which starts a `PostWorkflow`.
- Body: `{ "concept": "string" }`
- Response: `{ "postId": "uuid" }`
- Component: `PostEndpoint` → `ConceptQueue`

### POST /api/posts/{postId}/approve
Approve a post in `AWAITING_APPROVAL`.
- Body: `{ "approvedBy": "string", "comment": "string" }`
- Response: `200` | `404`
- Component: `PostEndpoint` → `PostEntity.approve`

### POST /api/posts/{postId}/reject
Reject a post in `AWAITING_APPROVAL`.
- Body: `{ "rejectedBy": "string", "reason": "string" }`
- Response: `200` | `404`
- Component: `PostEndpoint` → `PostEntity.reject`

### GET /api/posts
List all posts. Optional `?status=RESEARCHING|COMPOSING|AWAITING_APPROVAL|APPROVED|REJECTED|ESCALATED|PUBLISHED` filtered client-side in the endpoint (the view has no enum-indexed query).
- Response: `{ "posts": [PostBrief, ...] }`
- Component: `PostEndpoint` → `PostsView.getAllPosts`

### GET /api/posts/{postId}
One post.
- Response: `PostBrief` | `404`
- Component: `PostEndpoint` → `PostEntity.getPost`

### GET /api/posts/sse
Server-Sent Events stream of every post change.
- Response: `text/event-stream`, each `data:` line a `PostBrief` JSON object.
- Component: `PostEndpoint` → `PostsView` (SSE)

## Tool endpoints (WebTools, `/api/tools`)

In-process simulation of the web-search and page-browse tools the research agent calls. Both apply the G1 URL allowlist guard.

### POST /api/tools/search
- Body: `{ "query": "string" }`
- Response: `{ "results": [ { "title": "string", "url": "string", "snippet": "string" } ] }`
- Component: `WebTools` (canned from `sample-data/search-results.json`)

### POST /api/tools/browse
- Body: `{ "url": "string" }`
- Response: `{ "url": "string", "text": "string" }` — `403` if the host is outside the allowlist.
- Component: `WebTools` (canned from `sample-data/pages/`)

## Metadata endpoints (PostEndpoint, `/api/metadata`)

| Path | Response |
|---|---|
| `GET /api/metadata/eval-matrix` | `text/yaml` |
| `GET /api/metadata/risk-survey` | `text/yaml` |
| `GET /api/metadata/readme` | `text/markdown` |

Served from `src/main/resources/metadata/`.

## UI endpoints (AppEndpoint)

| Path | Response |
|---|---|
| `GET /` | `302 → /app/index.html` |
| `GET /app/{*path}` | static file from `static-resources/` |

## PostBrief JSON form

```json
{
  "id": "uuid",
  "concept": "string",
  "status": "RESEARCHING | COMPOSING | AWAITING_APPROVAL | APPROVED | REJECTED | ESCALATED | PUBLISHED",
  "researchedAt": "ISO-8601 or null",
  "researchNotes": "string or null",
  "composedAt": "ISO-8601 or null",
  "caption": "string or null",
  "hashtags": ["string", "..."],
  "visualBrief": "string or null",
  "brandCheckPassed": "boolean or null",
  "brandCheckNotes": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverComment": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "escalatedAt": "ISO-8601 or null",
  "publishedAt": "ISO-8601 or null",
  "publishedPermalink": "string or null"
}
```

`hashtags` is an empty array (never null) before the compose step.
