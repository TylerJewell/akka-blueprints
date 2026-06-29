# API contract — story-teller

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/stories` | `SubmitStoryRequest` | `201 { storyId }` | `StoryEndpoint` → `StoryEntity` |
| `GET` | `/api/stories` | — | `200 [ Story... ]` (newest-first) | `StoryEndpoint` ← `StoryView` |
| `GET` | `/api/stories/{id}` | — | `200 Story` / `404` | `StoryEndpoint` ← `StoryView` |
| `GET` | `/api/stories/sse` | — | `text/event-stream` | `StoryEndpoint` ← `StoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitStoryRequest (request body)

```json
{
  "genre": "Mystery",
  "rawPrompt": "A detective finds a locked room with no doors.",
  "constraints": {
    "tone": "GRITTY",
    "wordCountTarget": 300,
    "pointOfView": "FIRST"
  },
  "requestedBy": "author-session-42"
}
```

### Story (response body)

```json
{
  "storyId": "s-7a4...",
  "request": {
    "storyId": "s-7a4...",
    "genre": "Mystery",
    "rawPrompt": "A detective finds a locked room with no doors.",
    "constraints": {
      "tone": "GRITTY",
      "wordCountTarget": 300,
      "pointOfView": "FIRST"
    },
    "requestedBy": "author-session-42",
    "requestedAt": "2026-06-28T12:00:00Z"
  },
  "enriched": {
    "safePrompt": "A detective finds a locked room with no doors.",
    "contentTags": ["mystery", "crime", "locked-room"],
    "safetyPassed": true,
    "safetyReason": null
  },
  "story": {
    "title": "The Room That Wasn't There",
    "body": "I measured the wall three times...\n\nSomething had been here before me.",
    "authorNote": "I kept the viewpoint tightly inside the detective's sensory experience so the impossibility accumulates through touch and smell.",
    "genre": "Mystery",
    "generatedAt": "2026-06-28T12:00:22Z"
  },
  "quality": {
    "score": 4,
    "rationale": "Body meets word count and has paragraph breaks; author's note is informative.",
    "evaluatedAt": "2026-06-28T12:00:23Z"
  },
  "status": "SCORED",
  "createdAt": "2026-06-28T12:00:00Z",
  "finishedAt": "2026-06-28T12:00:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### Blocked story (response body)

When a prompt fails the content-safety check, the response looks like:

```json
{
  "storyId": "s-8b5...",
  "request": { "..." : "..." },
  "enriched": {
    "safePrompt": null,
    "contentTags": [],
    "safetyPassed": false,
    "safetyReason": "Prompt contains disallowed content category: explicit-violence"
  },
  "story": null,
  "quality": null,
  "status": "BLOCKED",
  "createdAt": "2026-06-28T12:01:00Z",
  "finishedAt": "2026-06-28T12:01:00Z"
}
```

### SSE event format

```
event: story-update
data: { "storyId": "s-7a4...", "status": "STORY_RECORDED", "story": { ... }, ... }
```

One event per state transition (`REQUESTED`, `ENRICHED`, `BLOCKED`, `GENERATING`, `STORY_RECORDED`, `SCORED`, `FAILED`). Clients reconcile by `storyId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
