# API contract — AI Blog Writer Pipeline with Ollama

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/posts` | `SubmitPostRequest` | `201 { postId }` | `PostEndpoint` → `PostEntity` |
| `GET` | `/api/posts` | — | `200 [ PostRecord... ]` (newest-first) | `PostEndpoint` ← `PostView` |
| `GET` | `/api/posts/{id}` | — | `200 PostRecord` / `404` | `PostEndpoint` ← `PostView` |
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
  "topic": "Getting started with reactive systems",
  "postType": "tutorial"
}
```

`postType` accepted values: `"tutorial"`, `"explainer"`, `"opinion"`.

### PostRecord (response body)

```json
{
  "postId": "p-3bc7a1...",
  "topic": "Getting started with reactive systems",
  "postType": "tutorial",
  "research": {
    "references": [
      {
        "source": "Reactive Manifesto",
        "url": "https://example.org/reactive-manifesto",
        "summary": "Defines the four tenets of reactive systems: responsive, resilient, elastic, and message-driven.",
        "fetchedAt": "2026-06-28T10:00:00Z"
      }
    ],
    "topicSummary": "Reactive systems are architectural patterns for building responsive, resilient, and scalable services.",
    "researchedAt": "2026-06-28T10:00:00Z"
  },
  "outline": {
    "title": "Getting started with reactive systems",
    "sections": [
      {
        "sectionId": "what-is-reactive",
        "heading": "What makes a system reactive",
        "keyPoints": ["Four tenets from the Reactive Manifesto", "Message-driven as the enabling mechanism"]
      }
    ],
    "outlinedAt": "2026-06-28T10:00:05Z"
  },
  "draft": {
    "title": "Getting started with reactive systems",
    "introduction": "Reactive systems address the reliability and scalability challenges of modern distributed applications.",
    "sections": [
      {
        "sectionId": "what-is-reactive",
        "heading": "What makes a system reactive",
        "body": "The Reactive Manifesto defines four tenets: responsive, resilient, elastic, and message-driven. ..."
      }
    ],
    "draftedAt": "2026-06-28T10:00:10Z"
  },
  "editedDraft": {
    "title": "Getting started with reactive systems",
    "introduction": "Reactive systems address the reliability and scalability challenges of modern distributed applications.",
    "sections": [
      {
        "sectionId": "what-is-reactive",
        "heading": "What makes a system reactive",
        "body": "The Reactive Manifesto defines four core properties ...",
        "readability": { "fleschKincaid": 52, "toneLabel": "technical-neutral" }
      }
    ],
    "editedAt": "2026-06-28T10:00:15Z"
  },
  "post": {
    "title": "Getting started with reactive systems",
    "summary": "A practical introduction to the four tenets of reactive architecture and the actor model.",
    "postType": "tutorial",
    "sections": [
      {
        "sectionId": "what-is-reactive",
        "heading": "What makes a system reactive",
        "body": "The Reactive Manifesto defines four core properties ...\n\n```java\n// example actor\n```",
        "references": [
          { "label": "Reactive Manifesto", "url": "https://example.org/reactive-manifesto" }
        ]
      }
    ],
    "publishedAt": "2026-06-28T10:00:20Z"
  },
  "lastPolicyCheck": {
    "passed": true,
    "reason": "All checks passed.",
    "checkedAt": "2026-06-28T10:00:20Z"
  },
  "status": "PUBLISHED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:20Z",
  "guardrailBlocks": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailBlocks` is an array — empty on the happy path, populated when the guardrail blocked a non-compliant response.

### SSE event format

```
event: post-update
data: { "postId": "p-3bc7a1...", "status": "DRAFTED", "draft": { ... }, ... }
```

One event per state transition (`CREATED`, `RESEARCHING`, `RESEARCHED`, `OUTLINING`, `OUTLINED`, `DRAFTING`, `DRAFTED`, `EDITING`, `EDITED`, `PUBLISHING`, `PUBLISHED`, `FAILED`) and one per `GuardrailBlocked` audit event:

```
event: post-blocked
data: { "postId": "p-3bc7a1...", "phase": "DRAFT", "reason": "policy-violation: prohibited keyword 'X' found in prose", "checkedAt": "..." }
```

Clients reconcile by `postId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitPostRequest` record and the `PostCreated` event to carry it.
