# API contract — gitwiki

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/updates` | `SubmitUpdateRequest` | `201 { updateId }` | `WikiEndpoint` → `PageEntity` + `WikiUpdateWorkflow` |
| `GET` | `/api/updates` | — | `200 [ UpdateRow... ]` (newest-first) | `WikiEndpoint` ← `PageView` |
| `GET` | `/api/updates/{id}` | — | `200 UpdateRow` / `404` | `WikiEndpoint` ← `PageView` |
| `GET` | `/api/updates/sse` | — | `text/event-stream` | `WikiEndpoint` ← `PageView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `WikiEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `WikiEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `WikiEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitUpdateRequest (request body)

```json
{
  "pagePath": "wiki/getting-started.md",
  "pageTitle": "Getting Started",
  "currentBody": "# Getting Started\n\nThis guide walks you through ...",
  "requestedChanges": "Add a Prerequisites section before the Installation section listing Java 21 and Maven 3.9.",
  "author": "dana-contributors"
}
```

### UpdateRow (response body — full detail)

```json
{
  "updateId": "u-7c3a...",
  "pagePath": "wiki/getting-started.md",
  "pageTitle": "Getting Started",
  "author": "dana-contributors",
  "draft": {
    "editedBody": "# Getting Started\n\n## Prerequisites\n\n- Java 21\n- Maven 3.9\n\n## Installation\n...",
    "commitMessage": "Add Prerequisites section to Getting Started",
    "diffSummary": "Added Prerequisites section listing Java 21 and Maven 3.9 before Installation."
  },
  "pushResult": {
    "status": "SUCCESS",
    "commitSha": "a3f9b2c1d4e5f678...",
    "rejectionReason": null,
    "pushedAt": "2026-06-28T14:22:31Z"
  },
  "status": "PUSHED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:31Z"
}
```

A `PUSH_REJECTED` row:

```json
{
  "updateId": "u-9e1f...",
  "pagePath": "wiki/architecture-overview.md",
  "pageTitle": "Architecture Overview",
  "author": "pat-reviewer",
  "draft": {
    "editedBody": "...",
    "commitMessage": "Update diagram caption in Architecture Overview",
    "diffSummary": "Updated diagram caption to reflect v2 component names."
  },
  "pushResult": {
    "status": "REJECTED",
    "commitSha": null,
    "rejectionReason": "target branch 'main' is protected; configured branch is 'wiki-drafts'",
    "pushedAt": "2026-06-28T14:23:10Z"
  },
  "status": "PUSH_REJECTED",
  "createdAt": "2026-06-28T14:23:00Z",
  "finishedAt": "2026-06-28T14:23:10Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: update-event
data: { "updateId": "u-7c3a...", "status": "PUSHED", "pushResult": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `EDITING`, `COMMIT_READY`, `PUSH_IN_PROGRESS`, `PUSHED`, `PUSH_REJECTED`, `CONFLICT`, `FAILED`). Clients reconcile by `updateId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay history.

## Authorization

ACL: open to the local runtime (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `author` from the authenticated principal rather than the request body.
