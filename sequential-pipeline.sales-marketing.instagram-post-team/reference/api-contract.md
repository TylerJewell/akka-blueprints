# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Timestamps are ISO-8601 strings; lifecycle fields are nullable (`Optional<T>` in Java, `null` on the wire).

## Pipeline endpoints (PostEndpoint, `/api`)

### POST /api/briefs
Submit a creative brief. Enqueues onto `BriefQueue`, which starts a `PostWorkflow`.
- Body: `{ "brief": "string" }`
- Response: `{ "postId": "uuid" }`
- Component: `PostEndpoint` → `BriefQueue`

### GET /api/posts
List all posts. Optional `?status=COMPOSING|PROMPTING|READY|BLOCKED|FAILED` filtered client-side in the endpoint (the view has no enum-indexed query).
- Response: `{ "posts": [PostContent, ...] }`
- Component: `PostEndpoint` → `PostsView.getAllPosts`

### GET /api/posts/{postId}
One post.
- Response: `PostContent` | `404`
- Component: `PostEndpoint` → `PostEntity.getPost`

### GET /api/posts/sse
Server-Sent Events stream of every post change.
- Response: `text/event-stream`, each `data:` line a `PostContent` JSON object.
- Component: `PostEndpoint` → `PostsView` (SSE)

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

## PostContent JSON form

```json
{
  "id": "uuid",
  "brief": "string",
  "status": "COMPOSING | PROMPTING | READY | BLOCKED | FAILED",
  "composedAt": "ISO-8601 or null",
  "caption": "string or null",
  "hashtags": ["string", "..."],
  "promptedAt": "ISO-8601 or null",
  "imagePrompt": "string or null",
  "safetyCheckPassed": "boolean or null",
  "safetyCheckNotes": "string or null",
  "readyAt": "ISO-8601 or null",
  "blockedAt": "ISO-8601 or null",
  "blockReason": "string or null"
}
```

`hashtags` is an empty array (never null) before the caption step.

## SSE event format

Each post change is emitted as one SSE record:

```
data: {"id":"...","brief":"...","status":"READY", ...}

```

The client parses each `data:` line as a `PostContent` JSON object and upserts it into the list by `id`.
