# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/publish-request` | `{ "topic": "string" }` | `{ "postId": "uuid" }` | PublishingEndpoint → PublishingWorkflow |
| POST | `/api/posts/{postId}/approve` | `{ "approvedBy": "string", "comment": "string" }` | `200` \| `404` | PublishingEndpoint → PostEntity |
| POST | `/api/posts/{postId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | PublishingEndpoint → PostEntity |
| GET | `/api/posts` | — | `{ "posts": [Post, ...] }` | PublishingEndpoint → PostsView |
| GET | `/api/posts/{postId}` | — | `Post` \| `404` | PublishingEndpoint → PostEntity |
| GET | `/api/posts/sse` | — | SSE stream of `Post` | PublishingEndpoint → PostsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | PublishingEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | PublishingEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | PublishingEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Post` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "topic": "string or null",
  "status": "DRAFTED | APPROVED | REJECTED | PUBLISHED",
  "draftedAt": "ISO-8601 or null",
  "draftTitle": "string or null",
  "draftContent": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverComment": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "publishedAt": "ISO-8601 or null",
  "publishedUrl": "string or null"
}
```

## SSE event format

`GET /api/posts/sse` streams the `PostsView` rows. Each message:

```
event: post
data: { ...Post... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
