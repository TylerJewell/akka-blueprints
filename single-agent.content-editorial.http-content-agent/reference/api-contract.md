# API contract — http-content-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `SubmitJobRequest` | `201 { jobId }` | `ContentEndpoint` → `ContentJobEntity` |
| `GET` | `/api/jobs` | — | `200 [ ContentJob... ]` (newest-first) | `ContentEndpoint` ← `ContentView` |
| `GET` | `/api/jobs/{id}` | — | `200 ContentJob` / `404` | `ContentEndpoint` ← `ContentView` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` | `ContentEndpoint` ← `ContentView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitJobRequest (request body)

```json
{
  "topic": "How event-driven architecture reduces operational overhead",
  "targetAudience": "platform engineers at mid-size companies",
  "outputFormat": "BLOG_POST",
  "wordCountHint": 600,
  "submittedBy": "content-author-42"
}
```

`wordCountHint` is optional; defaults to 300 when omitted.
`outputFormat` must be one of `BLOG_POST`, `SOCIAL_POST`, `EMAIL_TEASER`, `PRODUCT_DESCRIPTION`.

### ContentJob (response body)

```json
{
  "jobId": "j-7ae...",
  "brief": {
    "topic": "How event-driven architecture reduces operational overhead",
    "targetAudience": "platform engineers at mid-size companies",
    "outputFormat": "BLOG_POST",
    "wordCountHint": 600,
    "submittedBy": "content-author-42",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "content": {
    "title": "Event-Driven Architecture: Less Ops, More Flow",
    "body": "For platform engineers managing a growing fleet of services, ...",
    "format": "BLOG_POST",
    "wordCount": 607,
    "generatedAt": "2026-06-28T14:00:18Z"
  },
  "status": "APPROVED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:18Z"
}
```

Any lifecycle field that has not yet occurred is `null` on the wire (Akka serialises
`Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: job-update
data: { "jobId": "j-7ae...", "status": "APPROVED", "content": { ... }, ... }
```

One event per state transition (`REQUESTED`, `GENERATING`, `APPROVED`, `FAILED`). Clients
reconcile by `jobId`; an event always carries the full row at the moment of transition, so a
late-joining client never needs to replay.

## Authorization

ACL: open (local-dev only). A deployer adding identity must wrap the endpoint with their auth
middleware and set `submittedBy` from the authenticated principal rather than the request body.
