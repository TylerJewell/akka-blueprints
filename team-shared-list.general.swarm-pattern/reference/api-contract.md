# API contract — swarm-pattern

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "title": String, "description": String, "submittedBy"?: String }` | `202 { "jobId": String }` | `SwarmEndpoint` → `SubmissionLog` |
| `GET` | `/api/jobs/{id}` | — | `200 Job` / `404` | `SwarmEndpoint` ← `JobEntity` |
| `GET` | `/api/items` | — | `200 [ WorkItem... ]` | `SwarmEndpoint` ← `WorkListView` |
| `GET` | `/api/items?jobId=…&status=…` | — | `200 [ WorkItem... ]` (filtered client-side) | `SwarmEndpoint` ← `WorkListView` |
| `GET` | `/api/items/sse` | — | `text/event-stream` (one event per item change) | `SwarmEndpoint` ← `WorkListView` |
| `GET` | `/api/mailbox/{workerId}` | — | `200 [ RelayMessage... ]` | `SwarmEndpoint` ← `RelayMailbox` |
| `POST` | `/api/mailbox/{workerId}/messages/{messageId}/reply` | `{ "reply": String }` | `200` / `404` | `SwarmEndpoint` → `RelayMailbox` |
| `POST` | `/api/control/pause` | `{ "reason": String, "by": String }` | `200 SwarmControl` | `SwarmEndpoint` → `SwarmControl` |
| `POST` | `/api/control/resume` | — | `200 SwarmControl` | `SwarmEndpoint` → `SwarmControl` |
| `GET` | `/api/control` | — | `200 SwarmControl` | `SwarmEndpoint` ← `SwarmControl` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Job

```json
{
  "jobId": "j-7a1…",
  "title": "Classify customer feedback",
  "description": "Classify and summarise this quarter's customer feedback.",
  "submittedBy": "ui",
  "status": "IN_PROGRESS",
  "itemIds": ["j-7a1-i0", "j-7a1-i1", "j-7a1-i2"],
  "planSummary": "Load, classify, tag, and aggregate feedback in dependency order.",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": null
}
```

### WorkItem

```json
{
  "itemId": "j-7a1-i1",
  "jobId": "j-7a1",
  "title": "Classify sentiment for each entry",
  "description": "Assign a positive, neutral, or negative label to each feedback entry.",
  "dependsOn": ["Load the feedback batch"],
  "status": "DONE",
  "claimedBy": "worker-2",
  "claimedAt": "2026-06-28T09:10:08Z",
  "resultSummary": "Sentiment classified across 847 feedback entries.",
  "outputCount": 1,
  "qualityReport": { "passed": true, "issues": [], "ranAt": "2026-06-28T09:10:21Z" },
  "stalledReason": null,
  "completedAt": "2026-06-28T09:10:21Z",
  "createdAt": "2026-06-28T09:10:03Z"
}
```

Lifecycle fields (`claimedBy`, `claimedAt`, `resultSummary`, `outputCount`, `qualityReport`, `stalledReason`, `completedAt`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### RelayMessage

```json
{
  "messageId": "r-44b…",
  "fromWorker": "worker-1",
  "toWorker": "worker-2",
  "itemId": "j-7a1-i3",
  "question": "What topic tags did you assign and what are their counts?",
  "sentAt": "2026-06-28T09:10:30Z",
  "reply": null,
  "repliedAt": null
}
```

### SwarmControl

```json
{ "paused": false, "pausedReason": null, "pausedBy": null, "pausedAt": null }
```

### SSE event format

```
event: item-update
data: { "itemId": "j-7a1-i1", "status": "IN_REVIEW", ... }
```

One event per item state transition. Clients reconcile by `itemId` and group the board by `status`.
