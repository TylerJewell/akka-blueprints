# API contract — shared-task-team

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/goals` | `{ "title": String, "description": String, "requestedBy"?: String }` | `202 { "goalId": String }` | `SharedTaskEndpoint` → `IntakeQueue` |
| `GET` | `/api/goals/{id}` | — | `200 Goal` / `404` | `SharedTaskEndpoint` ← `GoalEntity` |
| `GET` | `/api/tasks` | — | `200 [ Task... ]` | `SharedTaskEndpoint` ← `TaskBoardView` |
| `GET` | `/api/tasks?goalId=…&status=…` | — | `200 [ Task... ]` (filtered client-side) | `SharedTaskEndpoint` ← `TaskBoardView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `SharedTaskEndpoint` ← `TaskBoardView` |
| `GET` | `/api/mailbox/{workerId}` | — | `200 [ PeerMessage... ]` | `SharedTaskEndpoint` ← `WorkerMailbox` |
| `POST` | `/api/mailbox/{workerId}/messages/{messageId}/reply` | `{ "reply": String }` | `200` / `404` | `SharedTaskEndpoint` → `WorkerMailbox` |
| `POST` | `/api/control/halt` | `{ "reason": String, "by": String }` | `200 SystemControl` | `SharedTaskEndpoint` → `SystemControl` |
| `POST` | `/api/control/resume` | — | `200 SystemControl` | `SharedTaskEndpoint` → `SystemControl` |
| `GET` | `/api/control` | — | `200 SystemControl` | `SharedTaskEndpoint` ← `SystemControl` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Goal

```json
{
  "goalId": "g-7a1…",
  "title": "Cloud storage comparison report",
  "description": "A structured report comparing three cloud storage providers on cost, performance, and compliance.",
  "requestedBy": "ui",
  "status": "IN_PROGRESS",
  "taskIds": ["g-7a1-t0", "g-7a1-t1", "g-7a1-t2", "g-7a1-t3"],
  "planSummary": "Divide the report into parallel per-criterion sections then synthesize an executive summary.",
  "createdAt": "2026-06-28T09:10:00Z",
  "completedAt": null
}
```

### Task

```json
{
  "taskId": "g-7a1-t1",
  "goalId": "g-7a1",
  "title": "Research performance comparison",
  "description": "Benchmark read/write throughput and latency for the three providers using published data.",
  "dependsOn": [],
  "status": "DONE",
  "claimedBy": "worker-2",
  "claimedAt": "2026-06-28T09:10:08Z",
  "artifactSummary": "Latency and throughput benchmarks for three cloud storage providers from published data.",
  "sectionCount": 2,
  "qualityReport": { "passed": true, "failures": [], "checkedAt": "2026-06-28T09:10:21Z" },
  "blockedReason": null,
  "completedAt": "2026-06-28T09:10:21Z",
  "createdAt": "2026-06-28T09:10:04Z"
}
```

Lifecycle fields (`claimedBy`, `claimedAt`, `artifactSummary`, `sectionCount`, `qualityReport`, `blockedReason`, `completedAt`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### PeerMessage

```json
{
  "messageId": "m-3b8…",
  "fromWorker": "worker-3",
  "toWorker": "worker-2",
  "taskId": "g-7a1-t3",
  "question": "Has the performance comparison section been finalized so I can reference its conclusions in the executive summary?",
  "sentAt": "2026-06-28T09:10:15Z",
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
data: { "taskId": "g-7a1-t1", "status": "IN_REVIEW", ... }
```

One event per task state transition. Clients reconcile by `taskId` and group the board by `status`.
