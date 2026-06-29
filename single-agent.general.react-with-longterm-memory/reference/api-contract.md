# API contract — mem0-react-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `StartSessionRequest` | `201 { sessionId }` | `AgentEndpoint` → `SessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `AgentEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 SessionRow` / `404` | `AgentEndpoint` ← `SessionView` |
| `POST` | `/api/sessions/{id}/turns` | `PostTurnRequest` | `202 { turnId }` | `AgentEndpoint` → `SessionEntity`, `Mem0ReactAgent` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `AgentEndpoint` ← `SessionView` |
| `GET` | `/api/memory/{userId}` | — | `200 [ MemoryRow... ]` | `AgentEndpoint` ← `MemoryView` |
| `GET` | `/api/memory/sse` | — | `text/event-stream` | `AgentEndpoint` ← `MemoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body for `POST /api/sessions`)

```json
{
  "userId": "user-42"
}
```

### PostTurnRequest (request body for `POST /api/sessions/{id}/turns`)

```json
{
  "text": "I prefer metric units. What is 5 miles in kilometres?"
}
```

### SessionRow (response body)

```json
{
  "sessionId": "s-9a1...",
  "userId": "user-42",
  "turns": [
    {
      "sessionId": "s-9a1...",
      "userId": "user-42",
      "text": "I prefer metric units. What is 5 miles in kilometres?",
      "sentAt": "2026-06-28T10:00:00Z"
    }
  ],
  "latestAnswer": {
    "turnId": "t-b2c...",
    "answer": "5 miles is approximately 8.05 km. I've noted your preference for metric units.",
    "toolCalls": [
      {
        "toolName": "recall-memories",
        "input": "{ \"userId\": \"user-42\" }",
        "output": "[]",
        "calledAt": "2026-06-28T10:00:02Z"
      },
      {
        "toolName": "calculate",
        "input": "5 * 1.60934",
        "output": "8.0467",
        "calledAt": "2026-06-28T10:00:03Z"
      },
      {
        "toolName": "store-memory",
        "input": "{ \"userId\": \"user-42\", \"text\": \"User prefers metric units.\" }",
        "output": "{ \"factId\": \"f-d44...\" }",
        "calledAt": "2026-06-28T10:00:04Z"
      }
    ],
    "answeredAt": "2026-06-28T10:00:05Z"
  },
  "status": "ACTIVE",
  "createdAt": "2026-06-28T10:00:00Z",
  "closedAt": null
}
```

### MemoryRow (response body for `GET /api/memory/{userId}`)

```json
{
  "factId": "f-d44...",
  "userId": "user-42",
  "cleanText": "User prefers metric units.",
  "piiCategoriesFound": [],
  "status": "PERSISTED",
  "createdAt": "2026-06-28T10:00:01Z",
  "persistedAt": "2026-06-28T10:00:02Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format — sessions

```
event: session-update
data: { "sessionId": "s-9a1...", "status": "ACTIVE", "latestAnswer": { ... }, ... }
```

One event per turn state transition. Clients reconcile by `sessionId`.

### SSE event format — memory

```
event: memory-update
data: { "factId": "f-d44...", "userId": "user-42", "status": "PERSISTED", ... }
```

```
event: drift-signal
data: { "userId": "user-42", "factCount": 51, "threshold": 50, "signaledAt": "2026-06-28T10:05:00Z" }
```

Two event types on the memory SSE stream: `memory-update` for fact lifecycle transitions, and `drift-signal` when `DriftMonitor` fires.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `userId` from the authenticated principal rather than the request body.
