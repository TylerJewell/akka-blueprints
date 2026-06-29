# API contract — gitty

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/drafts` | `QueueDraftRequest` | `201 { draftId }` | `DraftEndpoint` → `DraftEntity` |
| `GET` | `/api/drafts` | — | `200 [ Draft... ]` (newest-first) | `DraftEndpoint` ← `DraftView` |
| `GET` | `/api/drafts/{id}` | — | `200 Draft` / `404` | `DraftEndpoint` ← `DraftView` |
| `GET` | `/api/drafts/sse` | — | `text/event-stream` | `DraftEndpoint` ← `DraftView` |
| `POST` | `/api/drafts/{id}/approve` | `ApproveRequest` | `204` | `DraftEndpoint` → `DraftWorkflow` |
| `POST` | `/api/drafts/{id}/discard` | `DiscardRequest` | `204` | `DraftEndpoint` → `DraftWorkflow` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### QueueDraftRequest (request body)

```json
{
  "url": "https://github.com/akka/akka/issues/4321",
  "contextNote": "We addressed this in v3; keep it short.",
  "queuedBy": "maintainer-99"
}
```

`url` is parsed by `DraftEndpoint` into `ThreadRef{repoSlug, threadNumber, kind}`. The parser accepts both `/issues/N` and `/pull/N` paths.

### ApproveRequest (request body)

```json
{
  "actorId": "maintainer-99",
  "editedText": "Thanks for the report! This was fixed in v3.1 — please upgrade and reopen if it persists."
}
```

`editedText` may be the original draft text or any text the maintainer wrote in the UI. It is stored as-is in `DraftApproved.action.editedText`.

### DiscardRequest (request body)

```json
{
  "actorId": "maintainer-99"
}
```

### Draft (response body)

```json
{
  "draftId": "d-7af...",
  "request": {
    "draftId": "d-7af...",
    "thread": {
      "url": "https://github.com/akka/akka/issues/4321",
      "repoSlug": "akka/akka",
      "threadNumber": 4321,
      "kind": "ISSUE"
    },
    "contextNote": "We addressed this in v3; keep it short.",
    "queuedBy": "maintainer-99",
    "queuedAt": "2026-06-28T09:00:00Z"
  },
  "thread": {
    "title": "StreamRef materializer leaks on ActorSystem restart",
    "body": "After restarting the ActorSystem during tests, StreamRef...",
    "comments": [
      "I reproduced this on akka 2.8.1 as well.",
      "Any update on this? Seeing it in production."
    ],
    "commentCount": 2,
    "fetchedAt": "2026-06-28T09:00:01Z"
  },
  "draft": {
    "text": "Thanks for the detailed report, #4321. This was resolved as part of the v3.0 ActorSystem lifecycle overhaul. Please try upgrading and reopen if the leak persists.",
    "tone": "closing",
    "references": ["#4321", "v3.0", "ActorSystem lifecycle"],
    "draftedAt": "2026-06-28T09:00:12Z"
  },
  "action": {
    "actorId": "maintainer-99",
    "editedText": "Thanks for the detailed report, #4321. This was resolved as part of the v3.0 ActorSystem lifecycle overhaul. Please try upgrading and reopen if the leak persists.",
    "actedAt": "2026-06-28T09:01:30Z"
  },
  "status": "APPROVED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:01:30Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: draft-update
data: { "draftId": "d-7af...", "status": "PENDING_REVIEW", "draft": { ... }, ... }
```

One event per state transition (`QUEUED`, `THREAD_FETCHED`, `DRAFTING`, `PENDING_REVIEW`, `APPROVED`, `DISCARDED`, `FAILED`). Clients reconcile by `draftId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `queuedBy` / `actorId` from the authenticated principal rather than the request body.
