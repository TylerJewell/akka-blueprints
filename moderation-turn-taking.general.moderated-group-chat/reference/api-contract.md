# API contract

Every endpoint the generated `ChatEndpoint` and `AppEndpoint` expose. All `/api/*` responses are JSON unless noted. ACL is open to the internet for local development only.

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/sessions` | `StartRequest` | `{ "sessionId": "uuid" }` | `ChatEndpoint` → `SessionRequestQueue` |
| GET | `/api/sessions` | — (optional `?status=`) | `{ "sessions": [ChatSession, ...] }` | `ChatEndpoint` → `SessionsView` |
| GET | `/api/sessions/{id}` | — | `ChatSession` or 404 | `ChatEndpoint` → `SessionsView` |
| GET | `/api/sessions/sse` | — | `text/event-stream` of `ChatSession` | `ChatEndpoint` → `SessionsView` |
| POST | `/api/system/halt` | — | `{ "halted": true }` | `ChatEndpoint` → `SystemControl` |
| POST | `/api/system/resume` | — | `{ "halted": false }` | `ChatEndpoint` → `SystemControl` |
| GET | `/api/system/status` | — | `{ "halted": bool }` | `ChatEndpoint` → `SystemControl` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ChatEndpoint` (classpath) |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ChatEndpoint` (classpath) |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ChatEndpoint` (classpath) |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static file | `AppEndpoint` (static-resources) |

`GET /api/sessions?status=...` filters client-side from `getAllSessions` — the view query carries no `WHERE status` clause because the runtime cannot auto-index the enum column (Lesson 2).

## Request payloads

```java
record StartRequest(
  String topic    // the subject of the group chat session
) {}
```

A well-formed `StartRequest` has a non-empty `topic`. The topic may be any string; the Orchestrator agent handles irresolvable topics by concluding `MAX_TURNS_REACHED` rather than by rejecting the request.

The system endpoints (`/api/system/halt`, `/api/system/resume`) take no body.

## Response payloads

### `ChatSession` (the view row, returned by list / single / SSE)

JSON form; `Optional<T>` fields serialize as the raw value or `null` (Akka's Jackson config unwraps `Optional`, Lesson 6):

```json
{
  "id": "uuid",
  "topic": "string or null",
  "status": "CREATED | CHATTING | CONCLUDED | ESCALATED",
  "currentTurn": 0,
  "turns": [
    {
      "turn": 1,
      "assistant": "RESEARCHER | CRITIC",
      "message": "string",
      "flagged": false,
      "at": "ISO-8601"
    }
  ],
  "terminationReason": "CONSENSUS | MAX_TURNS_REACHED | ESCALATED or null",
  "conclusionSummary": "string or null",
  "startedAt": "ISO-8601 or null",
  "concludedAt": "ISO-8601 or null",
  "escalatedAt": "ISO-8601 or null",
  "qualityScore": 0.0,
  "qualityNotes": "string or null"
}
```

`qualityScore` is `Optional<Double>` in Java and serializes as a number or `null`. `turns` is never null — an empty array before the first turn.

## SSE event format

`GET /api/sessions/sse` streams the `SessionsView` through `componentClient.forView().method(SessionsView::streamAllSessions).source()` mapped to Server-Sent Events. Each event's `data` field is one `ChatSession` JSON object (the same form above), emitted whenever any session's projected row changes:

```
event: session
data: {"id":"...","status":"CHATTING","currentTurn":3,"turns":[...], ...}

event: session
data: {"id":"...","status":"CONCLUDED","terminationReason":"CONSENSUS","conclusionSummary":"...", ...}
```

The browser keeps a map keyed by `id` and replaces a row on each event, so a session visibly advances turn by turn and then shows its concluded outcome and quality score.

## Metadata endpoints

The three `/api/metadata/*` endpoints read the corresponding file from `src/main/resources/metadata/` and return it with the matching content type (`text/yaml` or `text/markdown`). They power the Risk Survey, Eval Matrix, and Overview tabs without the browser needing to reach outside the service origin.
