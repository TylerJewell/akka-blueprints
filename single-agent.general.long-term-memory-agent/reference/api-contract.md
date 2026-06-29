# API contract — long-term-memory-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `ConversationEndpoint` → `SessionEntity` |
| `GET` | `/api/sessions?userId=` | — | `200 [ SessionRow... ]` (newest-first) | `ConversationEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 SessionRow` / `404` | `ConversationEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}/turns` | — | `text/event-stream` (turn updates) | `ConversationEndpoint` ← `SessionView` |
| `POST` | `/api/sessions/{id}/turns` | `SendTurnRequest` | `201 { turnId }` / `409 (session closed)` | `ConversationEndpoint` → `SessionEntity` + `MemoryExtractionWorkflow` |
| `GET` | `/api/memories?userId=` | — | `200 [ MemoryRow... ]` (newest-first) | `ConversationEndpoint` ← `MemoryView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (session transitions) | `ConversationEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `ConversationEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `ConversationEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `ConversationEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "userId": "user-42",
  "sessionLabel": "Project planning — Q3"
}
```

`sessionLabel` is optional. If omitted, the server generates a label from the creation timestamp.

### SendTurnRequest (request body)

```json
{
  "userId": "user-42",
  "message": "What were the action items we agreed on last time?"
}
```

### SessionRow (response body — GET /api/sessions/{id})

```json
{
  "sessionId": "s-7a3b...",
  "userId": "user-42",
  "sessionLabel": "Project planning — Q3",
  "turns": [
    {
      "turnId": "t-c91d...",
      "message": "What were the action items we agreed on last time?",
      "reply": "Based on what I recall from our previous session, the three action items were: ...",
      "sentAt": "2026-06-28T14:00:00Z",
      "repliedAt": "2026-06-28T14:00:04Z",
      "status": "REPLIED"
    }
  ],
  "status": "ACTIVE",
  "createdAt": "2026-06-28T13:55:00Z",
  "closedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### MemoryRow (response body — GET /api/memories)

```json
{
  "entryId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "user-42",
  "sanitizedText": "User prefers bullet-point summaries over paragraphs.",
  "piiCategoriesRedacted": [],
  "kind": "PREFERENCE",
  "relevanceScore": 0.92,
  "recordedAt": "2026-06-28T14:00:06Z",
  "expiresAt": null
}
```

### SSE event format — session turns (`/api/sessions/{id}/turns`)

```
event: turn-update
data: { "turnId": "t-c91d...", "status": "REPLIED", "reply": "Based on what I recall ...", "repliedAt": "2026-06-28T14:00:04Z" }
```

One event per turn status change (`PENDING` → `REPLIED` or `PENDING` → `FAILED`). Clients reconcile by `turnId`.

### SSE event format — session transitions (`/api/sessions/sse`)

```
event: session-update
data: { "sessionId": "s-7a3b...", "status": "CLOSED", "closedAt": "2026-06-28T15:30:00Z" }
```

One event per session status transition. Each event carries the full `SessionRow` at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `userId` from the authenticated principal rather than the request body.
