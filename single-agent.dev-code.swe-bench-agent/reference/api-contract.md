# API contract — swe-bench-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `SubmitTaskRequest` | `201 { taskId }` | `BenchmarkEndpoint` → `BenchmarkTaskEntity` |
| `GET` | `/api/tasks` | — | `200 [ BenchmarkTask... ]` (newest-first) | `BenchmarkEndpoint` ← `BenchmarkView` |
| `GET` | `/api/tasks/{id}` | — | `200 BenchmarkTask` / `404` | `BenchmarkEndpoint` ← `BenchmarkView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` | `BenchmarkEndpoint` ← `BenchmarkView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTaskRequest (request body)

```json
{
  "title": "Fix nil pointer dereference in HTTP handler",
  "description": "handler.go:42 panics when http.Get returns a nil response under certain transport configurations. The resp.Body.Close() call is reached before the nil check.",
  "language": "go",
  "repositoryName": "acme-fetch-lib",
  "snapshotBytes": "<base64-encoded tarball>"
}
```

### BenchmarkTask (response body)

```json
{
  "taskId": "t-7c4a...",
  "description": {
    "taskId": "t-7c4a...",
    "title": "Fix nil pointer dereference in HTTP handler",
    "description": "handler.go:42 panics when http.Get returns a nil response...",
    "language": "go",
    "repositoryName": "acme-fetch-lib"
  },
  "snapshot": {
    "snapshotRef": "3e2a1b9f...",
    "filePaths": ["handler.go", "handler_test.go", "go.mod", "go.sum"],
    "testFilePaths": ["handler_test.go"]
  },
  "patch": {
    "unifiedDiff": "--- a/handler.go\n+++ b/handler.go\n@@ -40,6 +40,9 @@...",
    "hunks": [
      {
        "filePath": "handler.go",
        "startLine": 40,
        "endLine": 44,
        "oldContent": "\tresp, err := http.Get(url)\n\tif err != nil { ... }",
        "newContent": "\tresp, err := http.Get(url)\n\tif err != nil { ... }\n\tif resp == nil || resp.Body == nil { ... }"
      }
    ],
    "confidenceScore": 88,
    "patchSummary": "Add nil check for resp and resp.Body before Close to prevent panic.",
    "patchedAt": "2026-06-28T14:22:00Z"
  },
  "gateReport": {
    "verdict": "PASS",
    "testCases": [
      { "testId": "TestFetchSuccess", "testFile": "handler_test.go", "outcome": "PASS", "failureMessage": null },
      { "testId": "TestFetchNilResponse", "testFile": "handler_test.go", "outcome": "PASS", "failureMessage": null }
    ],
    "totalTests": 2,
    "passedTests": 2,
    "runtimeMs": 42,
    "completedAt": "2026-06-28T14:22:03Z"
  },
  "status": "COMPLETED",
  "attemptNumber": 1,
  "createdAt": "2026-06-28T14:21:55Z",
  "finishedAt": "2026-06-28T14:22:03Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: task-update
data: { "taskId": "t-7c4a...", "status": "GATE_PASSED", "gateReport": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SNAPSHOT_PREPARED`, `PATCHING`, `PATCH_PRODUCED`, `GATE_PASSED`, `GATE_FAILED`, `COMPLETED`, `FAILED`). Clients reconcile by `taskId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and supply a principal identifier at submit time.
