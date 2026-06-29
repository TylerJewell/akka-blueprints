# API contract — akka-memory-first-coding-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/projects/init` | `{ "projectPath": String, "requestedBy"?: String }` | `202 { "projectId": String }` | `ProjectEndpoint` → `ProjectEntity` → `ResearchWorkflow` |
| `GET` | `/api/projects` | — | `200 [ ProjectRow... ]` | `ProjectEndpoint` ← `ProjectView` |
| `GET` | `/api/projects/{id}` | — | `200 Project` / `404` | `ProjectEndpoint` ← `ProjectEntity` |
| `GET` | `/api/projects/{id}/memory` | — | `200 ProjectMemory` / `404` | `ProjectEndpoint` ← `ProjectEntity` |
| `POST` | `/api/projects/{id}/edit` | `{ "instruction": String, "requestedBy"?: String }` | `202 { "sessionId": String }` | `ProjectEndpoint` → `ProjectEntity` → `SessionRequestConsumer` → `EditWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (optional `?projectId=` filter) | `ProjectEndpoint` ← `ProjectView` |
| `GET` | `/api/sessions/{id}` | — | `200 EditSession` / `404` | `ProjectEndpoint` ← `EditSessionEntity` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session or project change) | `ProjectEndpoint` ← `ProjectView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `ProjectEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `ProjectEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `ProjectEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Project (full form, returned by `GET /api/projects/{id}`)

```json
{
  "projectId": "proj-3a1…",
  "projectPath": "/workspace/sample-service",
  "status": "READY",
  "plan": {
    "filesToRead": ["pom.xml", "src/main/java/io/example/UserEntity.java"],
    "questions": ["What framework does the project use?", "What commands does UserEntity expose?"]
  },
  "memory": {
    "blocks": [
      {
        "name": "service-purpose",
        "content": "A Java Akka service exposing a user management domain. Built with Akka 3.6.0 and Maven.",
        "sourceFiles": ["pom.xml", "src/main/resources/application.conf"]
      },
      {
        "name": "entity-model",
        "content": "UserEntity is an event-sourced entity with commands createUser, getUser, updateEmail. Events: UserCreated, EmailUpdated.",
        "sourceFiles": ["src/main/java/io/example/UserEntity.java"]
      }
    ],
    "systemPrompt": "You are working on a Java Akka 3.6.0 service in package io.example. The primary entity is UserEntity with event-sourced state. Apply Akka patterns for all state transitions.",
    "builtAt": "2026-06-28T09:15:00Z"
  },
  "failureReason": null,
  "createdAt": "2026-06-28T09:12:00Z",
  "readyAt": "2026-06-28T09:15:00Z"
}
```

### EditSession (full form, returned by `GET /api/sessions/{id}`)

```json
{
  "sessionId": "sess-8b4…",
  "projectId": "proj-3a1…",
  "instruction": "Add input validation to getUser — return 400 if id is blank.",
  "status": "COMPLETED",
  "patchPlan": {
    "edits": [
      {
        "filePath": "/workspace/src/main/java/io/example/UserEndpoint.java",
        "kind": "REPLACE",
        "patch": "--- ...\n+++ ...\n@@ -22,3 +22,5 @@\n-    public User getUser(String id) {\n+    public User getUser(String id) {\n+        if (id == null || id.isBlank()) throw new IllegalArgumentException(\"id required\");\n"
      }
    ],
    "rationale": "Adds a blank-check guard at the top of getUser."
  },
  "patchLog": [
    {
      "attempt": 1,
      "edit": { "filePath": "/workspace/src/main/java/io/example/UserEndpoint.java", "kind": "REPLACE", "patch": "..." },
      "verdict": "APPLIED",
      "result": { "filePath": "/workspace/src/main/java/io/example/UserEndpoint.java", "ok": true, "diff": "--- ...\n+++ ...\n@@ ... @@\n-    public User getUser...", "errorReason": null },
      "testResult": { "passed": true, "total": 12, "failed": 0, "output": null },
      "blocker": null,
      "recordedAt": "2026-06-28T09:22:05Z"
    }
  ],
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:21:55Z",
  "finishedAt": "2026-06-28T09:22:10Z"
}
```

### ProjectRow (list form, returned by `GET /api/projects`)

Same fields as `Project` but `memory.blocks` is replaced by `memoryBlockCount: int` and `systemPromptPreview: String` (first 200 chars of the prompt). The UI fetches the full project by id on expand.

### SessionRow (list form, returned by `GET /api/sessions`)

Same fields as `EditSession` but `patchLog` is truncated to the last 3 entries plus `truncatedFromTotal: int`. The UI fetches the full session by id on expand.

### SSE event format

```
event: project-update
data: { "projectId": "proj-3a1…", "status": "READY", "memoryBlockCount": 5 }
```

```
event: session-update
data: { "sessionId": "sess-8b4…", "projectId": "proj-3a1…", "status": "APPLYING", "patchCount": 2 }
```

```
event: control-update
data: { "halted": true, "reason": "reviewing suspicious DELETE patch", "haltedAt": "2026-06-28T09:25:00Z" }
```

One event per state transition. Clients reconcile by `projectId` or `sessionId`.
