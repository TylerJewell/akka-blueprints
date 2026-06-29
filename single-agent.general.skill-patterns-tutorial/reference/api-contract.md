# API contract — skill-patterns-tutorial

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/skill-runs` | `InvokeSkillRequest` | `201 { runId }` | `SkillRunEndpoint` → `SkillRunEntity` + `SkillRunWorkflow` |
| `GET` | `/api/skill-runs` | — | `200 [ SkillRunRow... ]` (newest-first) | `SkillRunEndpoint` ← `SkillRunView` |
| `GET` | `/api/skill-runs/{id}` | — | `200 SkillRunRow` / `404` | `SkillRunEndpoint` ← `SkillRunView` |
| `GET` | `/api/skill-runs/sse` | — | `text/event-stream` | `SkillRunEndpoint` ← `SkillRunView` |
| `POST` | `/api/skills` | `SkillDefinition` | `201 { skillId }` | `SkillRunEndpoint` (in-memory registry) |
| `GET` | `/api/skills` | — | `200 [ SkillDefinition... ]` | `SkillRunEndpoint` (in-memory registry) |
| `GET` | `/api/tool-stub/lookup` | — (query param `key`) | `200 { key, value, source }` | `SkillToolStub` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `SkillRunEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `SkillRunEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `SkillRunEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### InvokeSkillRequest (POST /api/skill-runs body)

```json
{
  "pattern": "INLINE",
  "patternInputJson": "{\"topic\": \"event sourcing\", \"style\": \"concise\"}",
  "requestedBy": "developer-01"
}
```

`pattern` is one of `INLINE`, `FILE_BASED`, `EXTERNAL`, `META_CREATOR`.

`patternInputJson` is a JSON-encoded string whose shape depends on `pattern`:

- `INLINE`: `{ "topic": "<string>", "style": "concise"|"detailed"|"bullet-list" }`
- `FILE_BASED`: `{ "textToSummarize": "<string>" }`
- `EXTERNAL`: `{ "lookupKey": "<string>" }`
- `META_CREATOR`: `{ "skillName": "<string>", "skillDescription": "<string>" }`

### SkillRunRow (GET /api/skill-runs, GET /api/skill-runs/{id} response)

```json
{
  "runId": "run-4c9a...",
  "request": {
    "runId": "run-4c9a...",
    "pattern": "EXTERNAL",
    "patternInputJson": "{\"lookupKey\": \"gamma\"}",
    "requestedBy": "developer-01",
    "requestedAt": "2026-06-28T14:00:00Z"
  },
  "result": {
    "patternName": "External",
    "output": "The lookup for key 'gamma' returned 'gamma phrase' from the in-process stub.",
    "wiringNote": "External skill: agent called the lookup-tool (ToolDefinition.http) targeting SkillToolStub at /api/tool-stub/lookup; the tool response was incorporated into this output.",
    "completedAt": "2026-06-28T14:00:22Z"
  },
  "failureReason": null,
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:22Z"
}
```

Lifecycle fields that have not yet been populated are `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SkillDefinition (POST /api/skills body and GET /api/skills element)

```json
{
  "skillId": "haiku-generator",
  "displayName": "Haiku Generator",
  "systemPrompt": "## Role\n\nYou write haikus...",
  "createdBy": "SkillDemoAgent",
  "createdAt": "2026-06-28T14:05:10Z"
}
```

### SkillToolStub response (GET /api/tool-stub/lookup?key=gamma)

```json
{
  "key": "gamma",
  "value": "the third Greek letter, often used as a radiation type or a correction factor",
  "source": "in-process-stub"
}
```

Unknown keys return HTTP 200 with `"value": "no-data-for-key:gamma"`.

### SSE event format

```
event: skill-run-update
data: { "runId": "run-4c9a...", "status": "COMPLETED", "result": { ... }, ... }
```

One event per state transition (`REQUESTED`, `RUNNING`, `COMPLETED`, `FAILED`). Each event carries the full `SkillRunRow` at the moment of transition so a late-joining client never needs to replay prior events.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoints with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
