# API contract — multi-agent-handoff-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/tasks` | — | `200 [ Task... ]` (newest-first); supports optional `?domain=…&status=…` filtered client-side | `WorkflowEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 Task` / `404` | `WorkflowEndpoint` ← `TaskView` |
| `POST` | `/api/tasks` | `{ "requesterId": String, "title": String, "description": String, "preferredDomain": String }` | `201 { "taskId": String }` | `WorkflowEndpoint` → `TaskQueue` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` | `WorkflowEndpoint` ← `TaskView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingTask (request body for `POST /api/tasks`, minus `taskId` and `receivedAt`)

```json
{
  "requesterId": "user-441",
  "title": "Q2 churn cohort analysis",
  "description": "Break down monthly churn by signup-month cohort for Q2. The CSV has columns: cohort_month, users_start, churned. Highlight the worst three cohorts.",
  "preferredDomain": "auto"
}
```

### Task

```json
{
  "taskId": "tk-7c3f91d2",
  "incoming": {
    "taskId": "tk-7c3f91d2",
    "requesterId": "user-441",
    "title": "Q2 churn cohort analysis",
    "description": "Break down monthly churn by signup-month cohort for Q2...",
    "preferredDomain": "auto",
    "receivedAt": "2026-06-28T09:00:00Z"
  },
  "admission": {
    "admitted": true,
    "rejectionReasons": []
  },
  "routing": {
    "domain": "DATA_ANALYSIS",
    "confidence": "high",
    "reason": "Cohort breakdown from a tabular data source."
  },
  "result": {
    "headline": "April cohort has the highest churn rate at 14.2% based on description.",
    "body": "## Q2 Churn Cohort Summary\n\nBased on the column structure provided...",
    "format": "MARKDOWN_REPORT",
    "specialistTag": "data-analyst",
    "completedAt": "2026-06-28T09:00:22Z"
  },
  "blockedTool": null,
  "routingScore": {
    "score": 5,
    "rationale": "Domain right; reason names the cohort plus data-source signal.",
    "scoredAt": "2026-06-28T09:00:14Z"
  },
  "rejectionReason": null,
  "status": "COMPLETED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:23Z"
}
```

A tool-blocked task has `blockedTool.allowed = false`, `blockedTool.toolName` populated, `status = "TOOL_BLOCKED"`, `finishedAt = null`. A rejected task has `routing.domain = "UNROUTABLE"` (or failed admission), `rejectionReason` populated, `status = "REJECTED"`.

### RoutingDecision (returned by `RouterAgent`, embedded in `Task.routing`)

```json
{ "domain": "DATA_ANALYSIS" | "CONTENT_WRITING" | "CODE_REVIEW" | "UNROUTABLE",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### TaskResult (returned by any specialist, embedded in `Task.result`)

```json
{ "headline": "≤ 100 char summary of what was produced",
  "body": "three to five paragraphs or structured JSON",
  "format": "MARKDOWN_REPORT" | "STRUCTURED_JSON" | "INLINE_COMMENT" | "PLAIN_TEXT",
  "specialistTag": "data-analyst" | "content-writer" | "code-reviewer",
  "completedAt": "ISO-8601" }
```

### ToolCallRecord (embedded in `Task.blockedTool` when tool was blocked)

```json
{ "toolName": "fetch_external_url",
  "argsDigest": "a3f9c21b",
  "allowed": false,
  "blockReason": "tool not in allowed-tool registry",
  "calledAt": "ISO-8601" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `Task.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: task-update
data: { "taskId": "tk-…", "status": "ROUTED", … full Task JSON … }
```

One event per state transition on the `TaskView`. Clients reconcile by `taskId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/tasks`).
- `404` — unknown `taskId`.
- `409` — operation requested on a task in an incompatible status.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
