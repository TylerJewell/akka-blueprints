# API contract — video-to-blog-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/posts` | `SubmitPostRequest` | `201 { postId }` | `BlogEndpoint` → `BlogPostEntity` |
| `GET` | `/api/posts` | — | `200 [ BlogPostRecord... ]` (newest-first) | `BlogEndpoint` ← `BlogPostView` |
| `GET` | `/api/posts/{id}` | — | `200 BlogPostRecord` / `404` | `BlogEndpoint` ← `BlogPostView` |
| `GET` | `/api/posts/sse` | — | `text/event-stream` | `BlogEndpoint` ← `BlogPostView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPostRequest (request body)

```json
{
  "videoUrl": "https://www.youtube.com/watch?v=seed-akka-agents"
}
```

### BlogPostRecord (response body)

```json
{
  "postId": "p-3fa7b1...",
  "videoUrl": "https://www.youtube.com/watch?v=seed-akka-agents",
  "transcript": {
    "videoUrl": "https://www.youtube.com/watch?v=seed-akka-agents",
    "text": "Welcome to this overview of Akka agents...",
    "durationSeconds": 1200,
    "chapters": [
      { "title": "What are Akka agents", "startSeconds": 0, "endSeconds": 480 },
      { "title": "Building a sequential pipeline", "startSeconds": 480, "endSeconds": 1200 }
    ],
    "extractedAt": "2026-06-28T10:00:00Z"
  },
  "summary": {
    "keyPoints": [
      { "pointId": "p-3a1c9f4e", "text": "Akka agents run typed tasks with bounded iteration budgets.", "sourceStartSeconds": 60 }
    ],
    "sections": [
      { "sectionId": "s-0", "heading": "What are Akka agents", "pointIds": ["p-3a1c9f4e"] }
    ],
    "summarisedAt": "2026-06-28T10:00:05Z"
  },
  "draft": {
    "introduction": "Akka agents bring type-safe task execution to LLM-powered applications.",
    "sections": [
      { "sectionId": "s-0", "heading": "What are Akka agents", "body": "Akka agents run discrete typed tasks, each with a bounded iteration budget." }
    ],
    "draftedAt": "2026-06-28T10:00:10Z"
  },
  "post": {
    "title": "Getting started with Akka agents, mid-2026",
    "introduction": "Akka agents bring type-safe task execution to LLM-powered applications. This post walks through what they are and how to chain them into a sequential pipeline.",
    "sections": [
      {
        "sectionId": "s-0",
        "heading": "What are Akka agents",
        "body": "Akka agents run discrete typed tasks, each with a bounded iteration budget. This design prevents runaway loops while giving the agent enough room to self-correct on tool call errors.",
        "sourceTimestamps": [{ "label": "Introduction segment", "startSeconds": 60 }]
      }
    ],
    "conclusion": "Akka agents and sequential pipelines together let you build governed, auditable content workflows.",
    "wordCount": 142,
    "polishedAt": "2026-06-28T10:00:15Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Section parity, heading completeness, word count, and body completeness all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:15Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:15Z",
  "guardrailRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when the POLISH task produced a prohibited-content response that the guardrail rejected.

### SSE event format

```
event: post-update
data: { "postId": "p-3fa7b1...", "status": "SUMMARISED", "summary": { ... }, ... }
```

One event per state transition (`CREATED`, `TRANSCRIBING`, `TRANSCRIBED`, `SUMMARISING`, `SUMMARISED`, `DRAFTING`, `DRAFTED`, `POLISHING`, `POLISHED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: post-rejection
data: { "postId": "p-3fa7b1...", "phase": "POLISH", "reason": "content-violation: response contains prohibited phrase 'earn money fast'", "rejectedAt": "..." }
```

Clients reconcile by `postId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitPostRequest` record and the `PostCreated` event to carry it.
