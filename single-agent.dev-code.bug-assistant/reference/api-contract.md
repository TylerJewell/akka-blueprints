# API contract — bug-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/bugs` | `SubmitBugRequest` | `201 { bugId }` | `BugEndpoint` → `BugEntity` |
| `GET` | `/api/bugs` | — | `200 [ Bug... ]` (newest-first) | `BugEndpoint` ← `BugView` |
| `GET` | `/api/bugs/{id}` | — | `200 Bug` / `404` | `BugEndpoint` ← `BugView` |
| `GET` | `/api/bugs/sse` | — | `text/event-stream` | `BugEndpoint` ← `BugView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitBugRequest (request body)

```json
{
  "title": "NullPointerException in UserService.findUser when userId is empty string",
  "description": "When the frontend passes an empty userId to the GET /users endpoint, UserService.findUser receives an empty string and throws NPE on the repository lookup. Stack trace: ...",
  "priority": "HIGH",
  "component": "user-service",
  "reportedBy": "eng-alice"
}
```

`priority` must be one of: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.

### Bug (response body)

```json
{
  "bugId": "b-7a3f...",
  "report": {
    "bugId": "b-7a3f...",
    "title": "NullPointerException in UserService.findUser when userId is empty string",
    "description": "When the frontend passes an empty userId ...",
    "priority": "HIGH",
    "component": "user-service",
    "reportedBy": "eng-alice",
    "reportedAt": "2026-06-28T09:10:00Z"
  },
  "ticket": {
    "ticketKey": "PROJ-1042",
    "assignee": "eng-bob",
    "labels": ["priority-high", "user-service", "bug"],
    "project": "PROJ",
    "fetchedAt": "2026-06-28T09:10:01Z"
  },
  "resolution": {
    "status": "FIXED",
    "resolutionBody": "Root cause: findUser does not guard against empty-string input before the repository lookup. Fix: add a precondition check — if userId is blank, throw IllegalArgumentException before the lookup.",
    "confidenceLevel": "HIGH",
    "evidence": [
      {
        "query": "Java NullPointerException empty string guard pattern",
        "source": "web",
        "snippet": "Use Preconditions.checkArgument(!userId.isEmpty(), ...) or Optional.ofNullable to guard against empty inputs before repository calls.",
        "url": "https://example.com/java-npe-guard"
      }
    ],
    "resolvedAt": "2026-06-28T09:10:22Z"
  },
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": "2026-06-28T09:10:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: bug-update
data: { "bugId": "b-7a3f...", "status": "RESOLVED", "resolution": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `ENRICHED`, `INVESTIGATING`, `RESOLVED`, `FAILED`). Clients reconcile by `bugId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Tool call trace

The `BugView` row includes a `toolCallTrace: List<ToolCallEntry>` field populated from the agent's tool invocations during the task. Each entry carries:

```json
{
  "tool": "search_web",
  "arguments": { "query": "Java NullPointerException empty string guard pattern" },
  "result": "...",
  "outcome": "PASSED",
  "calledAt": "2026-06-28T09:10:10Z"
}
```

For `write_ticket` calls that were rejected by the guardrail, `outcome` is `REJECTED` and `result` contains the guardrail's structured error message. The trace is visible in the App UI's selected-bug detail pane.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `reportedBy` from the authenticated principal rather than the request body.
