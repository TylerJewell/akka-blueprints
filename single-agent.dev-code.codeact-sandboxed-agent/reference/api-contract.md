# API contract — codeact-sandboxed-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `SubmitTaskRequest` | `201 { taskId }` | `TaskEndpoint` → `TaskEntity` |
| `GET` | `/api/tasks` | — | `200 [ Task... ]` (newest-first) | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 Task` / `404` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTaskRequest (request body)

```json
{
  "description": "Count word frequencies in the provided text and return a JSON object mapping each word to its count.",
  "contextData": "the quick brown fox jumps over the lazy dog the fox",
  "acceptanceCriterion": "output is a valid JSON object; 'the' maps to 3, 'fox' maps to 2",
  "submittedBy": "dev-user-42"
}
```

### Task (response body — after SOLVED)

```json
{
  "taskId": "t-9af...",
  "definition": {
    "taskId": "t-9af...",
    "description": "Count word frequencies in the provided text and return a JSON object mapping each word to its count.",
    "contextData": "the quick brown fox jumps over the lazy dog the fox",
    "acceptanceCriterion": "output is a valid JSON object; 'the' maps to 3, 'fox' maps to 2",
    "submittedBy": "dev-user-42",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "codeHistory": [
    {
      "iterationNumber": 1,
      "language": "python",
      "code": "import sys, json, collections\ntext = sys.stdin.read()\ncounts = collections.Counter(text.lower().split())\nprint(json.dumps(dict(counts)))",
      "generatedAt": "2026-06-28T10:00:05Z"
    }
  ],
  "outputHistory": [
    {
      "iterationNumber": 1,
      "sanitizedOutput": "{\"the\": 3, \"quick\": 1, \"brown\": 1, \"fox\": 2, \"jumps\": 1, \"over\": 1, \"lazy\": 1, \"dog\": 1}",
      "secretCategoriesFound": []
    }
  ],
  "resolution": {
    "status": "SOLVED",
    "finalOutput": "{\"the\": 3, \"quick\": 1, \"brown\": 1, \"fox\": 2, \"jumps\": 1, \"over\": 1, \"lazy\": 1, \"dog\": 1}",
    "iterationsUsed": 1,
    "resolvedAt": "2026-06-28T10:00:08Z"
  },
  "status": "SOLVED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:08Z"
}
```

### Task (response body — after HALTED)

```json
{
  "taskId": "t-3bc...",
  "definition": { "..." : "..." },
  "codeHistory": [
    {
      "iterationNumber": 1,
      "language": "python",
      "code": "import sys\ntoken = 'ghp_AbCdEfGhIjKlMnOpQrStUv012345'\nprint(f'token={token}')\n",
      "generatedAt": "2026-06-28T10:01:00Z"
    }
  ],
  "outputHistory": [],
  "resolution": null,
  "status": "HALTED",
  "createdAt": "2026-06-28T10:01:00Z",
  "finishedAt": "2026-06-28T10:01:03Z"
}
```

Note: `outputHistory` is empty for a halted task because the safety halt fires before `CodeExecuted` is written and before the `SecretSanitizer` runs. The raw output is not persisted.

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `codeHistory` and `outputHistory` are never `null` — they are empty arrays `[]` at `SUBMITTED`.

### SSE event format

```
event: task-update
data: { "taskId": "t-9af...", "status": "EXECUTING", "codeHistory": [...], "outputHistory": [...], ... }
```

One event per state transition (`SUBMITTED`, `EXECUTING`, `SOLVED`, `HALTED`, `FAILED`). Clients reconcile by `taskId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

For `EXECUTING` events, the event carries the latest `codeHistory` entry so the UI can show each code snippet and sanitized output as they arrive, without waiting for the terminal state.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
