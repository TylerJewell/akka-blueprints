# API Contract: Modular Agent Skills Loader

Base URL (local): `http://localhost:9958`

All request and response bodies are `application/json`. The SSE stream uses `text/event-stream`.

---

## Task Endpoints

### POST /tasks

Submit a new task. The endpoint creates a task record in `PENDING` state and dispatches it to `TaskDispatchAgent`.

**Owner**: `AgentOrchestrationEndpoint`

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| description | string | Yes | Human-readable description of the task |
| capability | string | Yes | Required capability (e.g. `"web-search"`) |
| requestedBy | string | No | Caller identifier for audit; defaults to `"anonymous"` |

```json
{
  "description": "Find the current price of crude oil",
  "capability": "web-search",
  "requestedBy": "user-42"
}
```

**Response — 202 Accepted**

| Field | Type | Description |
|---|---|---|
| taskId | string | Assigned task identifier |
| status | string | Always `"PENDING"` on initial response |
| dispatchedAt | string (ISO 8601) | Timestamp of task creation |

```json
{
  "taskId": "task-8f3a1c",
  "status": "PENDING",
  "dispatchedAt": "2026-06-29T10:00:00Z"
}
```

**Error codes**

| Code | Condition |
|---|---|
| 400 | Missing required field (`description` or `capability`) |
| 422 | Guardrail blocked the invocation — see `reason` field |
| 500 | Internal error in agent or workflow |

**422 body**

```json
{
  "taskId": "task-8f3a1c",
  "status": "REJECTED",
  "reason": "skill-disabled"
}
```

---

### GET /tasks/{taskId}

Get the current state of a task.

**Owner**: `AgentOrchestrationEndpoint` → `SkillExecutionView`

**Path parameter**: `taskId` (string)

**Response — 200 OK**

| Field | Type | Nullable | Description |
|---|---|---|---|
| taskId | string | No | Task identifier |
| skillId | string | Yes | Selected skill (null until SKILL_SELECTED) |
| status | string | No | Current `TaskStatus` value |
| capability | string | No | Requested capability |
| dispatchedAt | string | No | ISO 8601 |
| completedAt | string | Yes | ISO 8601; null until COMPLETED or REJECTED |

```json
{
  "taskId": "task-8f3a1c",
  "skillId": "web-search-v2",
  "status": "COMPLETED",
  "capability": "web-search",
  "dispatchedAt": "2026-06-29T10:00:00Z",
  "completedAt": "2026-06-29T10:00:04Z"
}
```

**Error codes**

| Code | Condition |
|---|---|
| 404 | Task not found |

---

### GET /tasks

SSE stream of task-status updates. The connection stays open; the server pushes one event per task-state transition.

**Owner**: `AgentOrchestrationEndpoint` → `SkillExecutionView`

**Query parameters**

| Parameter | Type | Description |
|---|---|---|
| capability | string | Filter to tasks with this capability (optional) |
| status | string | Filter to tasks with this status (optional) |

**SSE event format**

```
event: task-update
data: {"taskId":"task-8f3a1c","status":"GUARDRAIL_PASSED","skillId":"web-search-v2","updatedAt":"2026-06-29T10:00:02Z"}
```

| Field | Type | Description |
|---|---|---|
| taskId | string | Task that changed |
| status | string | New `TaskStatus` value |
| skillId | string | Selected skill (may be null for PENDING → SKILL_SELECTED transition) |
| updatedAt | string | ISO 8601 timestamp of the transition |

---

## Skill Endpoints

### GET /skills

List all registered skills.

**Owner**: `AgentOrchestrationEndpoint` → `SkillRegistryEntity`

**Response — 200 OK** — array of `SkillDescriptor`

```json
[
  {
    "skillId": "web-search-v2",
    "name": "Web Search",
    "version": "2.1.0",
    "capabilities": ["web-search", "url-fetch"],
    "enabled": true,
    "loadedAt": "2026-06-28T08:00:00Z"
  }
]
```

---

### POST /skills

Register a new skill or update an existing one by `skillId`.

**Owner**: `AgentOrchestrationEndpoint` → `SkillRegistryEntity`

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| skillId | string | Yes | Unique identifier |
| name | string | Yes | Display name |
| version | string | Yes | Semver string |
| capabilities | array of string | Yes | Capability tags |
| enabled | boolean | No | Defaults to `true` |

```json
{
  "skillId": "data-extract-v1",
  "name": "Data Extractor",
  "version": "1.0.0",
  "capabilities": ["data-extract", "csv-parse"],
  "enabled": true
}
```

**Response — 201 Created** — full `SkillDescriptor` including `loadedAt`

**Error codes**

| Code | Condition |
|---|---|
| 400 | Missing required field |
| 409 | skillId exists with same version and different capabilities (use new version) |

---

### PUT /skills/{skillId}/enable

Enable a previously disabled skill.

**Owner**: `AgentOrchestrationEndpoint` → `SkillRegistryEntity`

**Response — 204 No Content**

**Error codes**: 404 if skill not found.

---

### PUT /skills/{skillId}/disable

Disable a skill. Future guardrail checks for this skill will reject.

**Owner**: `AgentOrchestrationEndpoint` → `SkillRegistryEntity`

**Response — 204 No Content**

**Error codes**: 404 if skill not found.

---

### GET /skills/{skillId}

Get a single skill descriptor.

**Owner**: `AgentOrchestrationEndpoint` → `SkillRegistryEntity`

**Response — 200 OK** — `SkillDescriptor`

**Error codes**: 404 if skill not found.
