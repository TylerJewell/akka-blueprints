# API contract — autonomous-agent-pattern

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `SubmitTaskRequest` | `201 { taskId }` | `AgentTaskEndpoint` → `AgentTaskEntity` + `AgentTaskWorkflow` |
| `GET` | `/api/tasks` | — | `200 [ AgentTask... ]` (newest-first) | `AgentTaskEndpoint` ← `AgentTaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 AgentTask` / `404` | `AgentTaskEndpoint` ← `AgentTaskView` |
| `POST` | `/api/tasks/{id}/approve` | — | `200` / `409` (wrong status) | `AgentTaskEndpoint` → `AgentTaskEntity` |
| `POST` | `/api/tasks/{id}/reject` | — | `200` / `409` (wrong status) | `AgentTaskEndpoint` → `AgentTaskEntity` |
| `POST` | `/api/tasks/{id}/halt` | — | `200` / `409` (already terminal) | `AgentTaskEndpoint` → `AgentTaskEntity` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` | `AgentTaskEndpoint` ← `AgentTaskView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTaskRequest (request body)

```json
{
  "issueUrl": "https://github.com/acme/payments/issues/42",
  "submittedBy": "eng-platform-12"
}
```

The endpoint parses `issueUrl` into `IssueRef{repoOwner, repoName, issueNumber}` before writing to the entity. Any URL not matching `https://github.com/{owner}/{repo}/issues/{n}` returns `400`.

### AgentTask (response body — fully evaluated)

```json
{
  "taskId": "t-9f3a...",
  "issueRef": {
    "issueUrl": "https://github.com/acme/payments/issues/42",
    "repoOwner": "acme",
    "repoName": "payments",
    "issueNumber": 42,
    "submittedBy": "eng-platform-12",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "issueBody": {
    "title": "NPE in PaymentCalculator.compute() when discountCode is null",
    "body": "Stack trace: ...\nFailing test: ...",
    "labels": ["bug", "priority-high"],
    "fetchedAt": "2026-06-28T14:00:01Z"
  },
  "plan": {
    "summary": "Add a null guard in PaymentCalculator.compute() before the discountCode.toLowerCase() call.",
    "filesToChange": ["src/main/java/com/example/PaymentCalculator.java"],
    "approach": "Read the file, locate compute(), insert null guard, run mvn test.",
    "plannedAt": "2026-06-28T14:00:08Z"
  },
  "outcome": {
    "status": "SUCCEEDED",
    "patch": {
      "unifiedDiff": "--- a/src/main/java/com/example/PaymentCalculator.java\n+++ ...",
      "filesModified": ["src/main/java/com/example/PaymentCalculator.java"],
      "linesAdded": 2,
      "linesRemoved": 1
    },
    "confidenceScore": 0.92,
    "toolTrace": [
      {
        "toolCallId": "tc-01",
        "toolName": "readFile",
        "parameters": { "path": "src/main/java/com/example/PaymentCalculator.java" },
        "status": "SUCCEEDED",
        "result": "...",
        "blockReason": null,
        "calledAt": "2026-06-28T14:00:25Z"
      }
    ],
    "summary": "Added null guard. All 34 tests pass.",
    "completedAt": "2026-06-28T14:00:45Z"
  },
  "toolTrace": [ /* same as outcome.toolTrace, also exposed at top level */ ],
  "alerts": [],
  "status": "PATCH_READY",
  "haltRequested": false,
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:45Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### AgentTask — mid-execution snapshot

```json
{
  "taskId": "t-9f3a...",
  "issueRef": { "..." },
  "issueBody": null,
  "plan": null,
  "outcome": null,
  "toolTrace": [
    {
      "toolCallId": "tc-01",
      "toolName": "runShell",
      "parameters": { "command": "rm -rf /workspace" },
      "status": "BLOCKED",
      "result": null,
      "blockReason": "shell-danger-pattern: 'rm -rf' matches disallowed pattern",
      "calledAt": "2026-06-28T14:00:30Z"
    }
  ],
  "alerts": [],
  "status": "EXECUTING",
  "haltRequested": false,
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": null
}
```

### SSE event format

```
event: task-update
data: { "taskId": "t-9f3a...", "status": "CHECKPOINT_PENDING", "plan": { ... }, ... }
```

One event per state transition and per `ToolCallExecuted` / `MonitorAlertRaised` event. Clients reconcile by `taskId`; each SSE event carries the full row at the moment of the transition, so a late-joining client does not need to replay. High-frequency tool-call events during execution produce one SSE push per call — the UI throttles rendering client-side if needed.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and populate `submittedBy` from the authenticated principal rather than the request body. The `approve`, `reject`, and `halt` endpoints should additionally check that the caller has the appropriate role (reviewer or operator) before writing to the entity.
