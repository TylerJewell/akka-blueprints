# API contract — wp-autotagger

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/posts` | `SubmitPostRequest` | `201 { jobId }` | `PostEndpoint` → `PostEntity` |
| `GET` | `/api/posts` | — | `200 [ Post... ]` (newest-first) | `PostEndpoint` ← `PostView` |
| `GET` | `/api/posts/{id}` | — | `200 Post` / `404` | `PostEndpoint` ← `PostView` |
| `GET` | `/api/posts/sse` | — | `text/event-stream` | `PostEndpoint` ← `PostView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPostRequest (request body)

```json
{
  "postUrl": "https://myblog.example.com/?p=42",
  "config": {
    "maxTags": 8,
    "tagStyle": "MIXED",
    "submittedBy": "editor-jane"
  }
}
```

### Post (response body)

```json
{
  "jobId": "j-9c3...",
  "request": {
    "jobId": "j-9c3...",
    "postUrl": "https://myblog.example.com/?p=42",
    "config": {
      "maxTags": 8,
      "tagStyle": "MIXED",
      "submittedBy": "editor-jane"
    },
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "body": {
    "postId": "42",
    "title": "Getting started with async Python",
    "body": "Async programming in Python has become standard for I/O-bound workloads ...",
    "author": "Jane Smith",
    "publishedAt": "2026-06-27T08:30:00Z"
  },
  "proposal": {
    "tags": [
      { "tag": "python", "confidence": 1.0 },
      { "tag": "async programming", "confidence": 1.0 },
      { "tag": "asyncio", "confidence": 0.95 },
      { "tag": "concurrency", "confidence": 0.85 },
      { "tag": "io-bound workloads", "confidence": 0.8 }
    ],
    "tagStyle": "MIXED",
    "rationale": "Post covers async Python fundamentals using asyncio; concurrency and I/O workloads are the central use cases discussed.",
    "proposedAt": "2026-06-28T10:00:22Z"
  },
  "status": "TAGS_APPLIED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: post-update
data: { "jobId": "j-9c3...", "status": "TAGS_APPLIED", "proposal": { ... }, ... }
```

One event per state transition (`INGESTED`, `BODY_FETCHED`, `TAGGING`, `TAGS_APPLIED`, `FAILED`). Clients reconcile by `jobId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
