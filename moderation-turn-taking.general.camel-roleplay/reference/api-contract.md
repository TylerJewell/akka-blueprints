# API contract

Every endpoint the generated `CollaborationEndpoint` and `AppEndpoint` expose. All `/api/*` responses are JSON unless noted. ACL is open to the internet for local development only.

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/collaborations` | `StartRequest` | `{ "collaborationId": "uuid" }` | `CollaborationEndpoint` → `InboundTaskQueue` |
| GET | `/api/collaborations` | — (optional `?status=`) | `{ "collaborations": [Collaboration, ...] }` | `CollaborationEndpoint` → `CollaborationsView` |
| GET | `/api/collaborations/{id}` | — | `Collaboration` or 404 | `CollaborationEndpoint` → `CollaborationsView` |
| GET | `/api/collaborations/sse` | — | `text/event-stream` of `Collaboration` | `CollaborationEndpoint` → `CollaborationsView` |
| POST | `/api/system/halt` | — | `{ "halted": true }` | `CollaborationEndpoint` → `SystemControl` |
| POST | `/api/system/resume` | — | `{ "halted": false }` | `CollaborationEndpoint` → `SystemControl` |
| GET | `/api/system/status` | — | `{ "halted": bool }` | `CollaborationEndpoint` → `SystemControl` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `CollaborationEndpoint` (classpath) |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `CollaborationEndpoint` (classpath) |
| GET | `/api/metadata/readme` | — | `text/markdown` | `CollaborationEndpoint` (classpath) |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static file | `AppEndpoint` (static-resources) |

`GET /api/collaborations?status=...` filters client-side from `getAllCollaborations` — the view query carries no `WHERE status` clause because the runtime cannot auto-index the enum column (Lesson 2).

## Request payloads

```java
record StartRequest(
  String taskDescription,   // what the collaboration must solve
  String assistantRole,     // role label for the AI Assistant
  String userRole           // role label for the AI User
) {}
```

A well-formed `StartRequest` has non-empty `taskDescription`, `assistantRole`, and `userRole`. When the task has conflicting constraints that make convergence impossible, the collaboration is valid but will conclude `IMPASSE` — that is one of the acceptance journeys, not an error.

The system endpoints (`/api/system/halt`, `/api/system/resume`) take no body.

## Response payloads

### `Collaboration` (the view row, returned by list / single / SSE)

JSON form; `Optional<T>` fields serialize as the raw value or `null` (Akka's Jackson config unwraps `Optional`, Lesson 6):

```json
{
  "id": "uuid",
  "taskDescription": "string or null",
  "assistantRole": "string or null",
  "userRole": "string or null",
  "status": "CREATED | COLLABORATING | CONCLUDED | ESCALATED",
  "currentRound": 0,
  "turns": [
    {
      "round": 1,
      "agent": "ASSISTANT | USER",
      "message": "string",
      "intent": "string",
      "at": "ISO-8601"
    }
  ],
  "latestAssistantMessage": "string or null",
  "latestUserMessage": "string or null",
  "outcome": "SOLVED | IMPASSE or null",
  "solutionSummary": "string or null",
  "finalAnswer": "string or null",
  "startedAt": "ISO-8601 or null",
  "concludedAt": "ISO-8601 or null",
  "escalatedAt": "ISO-8601 or null",
  "outcomeScore": 0.0,
  "outcomeNotes": "string or null"
}
```

`outcomeScore` is `Optional<Double>` in Java and serializes as a number or `null`. `turns` is never null — an empty array before the first turn.

## SSE event format

`GET /api/collaborations/sse` streams the `CollaborationsView` through `componentClient.forView().method(CollaborationsView::streamAllCollaborations).source()` mapped to Server-Sent Events. Each event's `data` field is one `Collaboration` JSON object (the same form above), emitted whenever any collaboration's projected row changes:

```
event: collaboration
data: {"id":"...","status":"COLLABORATING","currentRound":2,"turns":[...], ...}

event: collaboration
data: {"id":"...","status":"CONCLUDED","outcome":"SOLVED","solutionSummary":"...", ...}
```

The browser keeps a map keyed by `id` and replaces a row on each event, so a collaboration visibly advances round by round and then shows its concluded outcome and evaluator score.

## Metadata endpoints

The three `/api/metadata/*` endpoints read the corresponding file from `src/main/resources/metadata/` and return it with the matching content type (`text/yaml` or `text/markdown`). They power the Risk Survey, Eval Matrix, and Overview tabs without the browser needing to reach outside the service origin.
