# API contract — dev-team-task-board

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/projects` | `{ "title": String, "description": String, "requestedBy"?: String }` | `202 { "projectId": String }` | `DevTeamEndpoint` → `IntakeQueue` |
| `GET` | `/api/projects/{id}` | — | `200 Project` / `404` | `DevTeamEndpoint` ← `ProjectEntity` |
| `GET` | `/api/tasks` | — | `200 [ Task... ]` | `DevTeamEndpoint` ← `TaskBoardView` |
| `GET` | `/api/tasks?projectId=…&status=…` | — | `200 [ Task... ]` (filtered client-side) | `DevTeamEndpoint` ← `TaskBoardView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `DevTeamEndpoint` ← `TaskBoardView` |
| `GET` | `/api/mailbox/{developerId}` | — | `200 [ PeerMessage... ]` | `DevTeamEndpoint` ← `PeerMailbox` |
| `POST` | `/api/mailbox/{developerId}/messages/{messageId}/reply` | `{ "reply": String }` | `200` / `404` | `DevTeamEndpoint` → `PeerMailbox` |
| `POST` | `/api/control/halt` | `{ "reason": String, "by": String }` | `200 SystemControl` | `DevTeamEndpoint` → `SystemControl` |
| `POST` | `/api/control/resume` | — | `200 SystemControl` | `DevTeamEndpoint` → `SystemControl` |
| `GET` | `/api/control` | — | `200 SystemControl` | `DevTeamEndpoint` ← `SystemControl` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Project

```json
{
  "projectId": "p-3f2…",
  "title": "URL shortener",
  "description": "An in-memory URL shortener with a lookup endpoint.",
  "requestedBy": "ui",
  "status": "IN_PROGRESS",
  "taskIds": ["p-3f2-t0", "p-3f2-t1", "p-3f2-t2"],
  "planSummary": "Build a single-service URL shortener around an in-memory store.",
  "createdAt": "2026-06-28T07:32:55Z",
  "completedAt": null
}
```

### Task

```json
{
  "taskId": "p-3f2-t1",
  "projectId": "p-3f2",
  "title": "Implement the in-memory store",
  "description": "A thread-safe map keyed by short code.",
  "dependsOn": ["Define the URL record"],
  "status": "DONE",
  "claimedBy": "dev-2",
  "claimedAt": "2026-06-28T07:33:01Z",
  "artifactSummary": "ConcurrentHashMap-backed store with put and get.",
  "fileCount": 1,
  "testReport": { "passed": true, "failures": [], "ranAt": "2026-06-28T07:33:14Z" },
  "blockedReason": null,
  "completedAt": "2026-06-28T07:33:14Z",
  "createdAt": "2026-06-28T07:32:58Z"
}
```

Lifecycle fields (`claimedBy`, `claimedAt`, `artifactSummary`, `fileCount`, `testReport`, `blockedReason`, `completedAt`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### PeerMessage

```json
{
  "messageId": "m-91a…",
  "fromDeveloper": "dev-1",
  "toDeveloper": "dev-2",
  "taskId": "p-3f2-t4",
  "question": "What is the method signature of the in-memory store's lookup by short code?",
  "sentAt": "2026-06-28T07:33:20Z",
  "reply": null,
  "repliedAt": null
}
```

### SystemControl

```json
{ "halted": false, "haltedReason": null, "haltedBy": null, "haltedAt": null }
```

### SSE event format

```
event: task-update
data: { "taskId": "p-3f2-t1", "status": "IN_REVIEW", ... }
```

One event per task state transition. Clients reconcile by `taskId` and group the board by `status`.
