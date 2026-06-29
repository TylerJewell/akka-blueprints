# API contract — with Signals & Queries

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `StartSessionRequest` | `201 { sessionId }` | `ChatEndpoint` → `ChatSessionEntity` + `ChatSessionWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `ChatEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 SessionState` / `404` | `ChatEndpoint` ← `ChatSessionEntity` |
| `PUT` | `/api/sessions/{id}/signal/turn` | `AddTurnSignal` | `204` / `400 SignalRejection` | `ChatEndpoint` → `ChatSessionWorkflow` |
| `PUT` | `/api/sessions/{id}/signal/pause` | `PauseSignal` | `204` | `ChatEndpoint` → `ChatSessionWorkflow` |
| `PUT` | `/api/sessions/{id}/signal/resume` | `ResumeSignal` | `204` | `ChatEndpoint` → `ChatSessionWorkflow` |
| `PUT` | `/api/sessions/{id}/signal/close` | — | `204` | `ChatEndpoint` → `ChatSessionWorkflow` |
| `GET` | `/api/sessions/{id}/query` | — | `200 SessionState` | `ChatEndpoint` → `ChatSessionWorkflow` (query, no event) |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `ChatEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body)

```json
{
  "sessionName": "Architecture Q&A — June 28",
  "initialPrompt": "What is the difference between a workflow step and a workflow signal?",
  "submittedBy": "user-42"
}
```

### AddTurnSignal (request body for `PUT /signal/turn`)

```json
{
  "prompt": "Can I send a signal to a workflow that has already closed?",
  "source": "USER"
}
```

### PauseSignal (request body for `PUT /signal/pause`)

```json
{
  "reason": "Agent is going off-topic; need to inject context before next turn."
}
```

### ResumeSignal (request body for `PUT /signal/resume`)

```json
{
  "correctiveNote": "Focus on Akka Workflow primitives only; do not discuss Spring or other frameworks."
}
```

### SessionRow (response body for `GET /api/sessions`)

```json
{
  "sessionId": "s-4a7b...",
  "sessionName": "Architecture Q&A — June 28",
  "turnCount": 3,
  "status": "ACTIVE",
  "lastReply": "Yes — signals can only be delivered to a running workflow instance...",
  "startedAt": "2026-06-28T14:00:00Z",
  "closedAt": null
}
```

### SessionState (response body for `GET /api/sessions/{id}` and `GET /api/sessions/{id}/query`)

```json
{
  "sessionId": "s-4a7b...",
  "sessionName": "Architecture Q&A — June 28",
  "turns": [
    {
      "turnId": "t-001",
      "prompt": "What is the difference between a workflow step and a workflow signal?",
      "reply": "A step is a named unit of work that executes once per activation...",
      "source": "USER",
      "sentAt": "2026-06-28T14:00:01Z",
      "repliedAt": "2026-06-28T14:00:08Z"
    },
    {
      "turnId": "t-002",
      "prompt": "Can I send a signal to a workflow that has already closed?",
      "reply": "No — once a workflow transitions to a terminal state...",
      "source": "USER",
      "sentAt": "2026-06-28T14:01:00Z",
      "repliedAt": "2026-06-28T14:01:06Z"
    }
  ],
  "status": "ACTIVE",
  "pauseReason": null,
  "startedAt": "2026-06-28T14:00:00Z",
  "closedAt": null
}
```

Any lifecycle field that has not occurred yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SignalRejection (400 response body for rejected `PUT /signal/turn`)

```json
{
  "rejectedCheck": "prompt-empty",
  "detail": "Prompt must be non-empty and non-whitespace."
}
```

Possible `rejectedCheck` values: `prompt-empty`, `prompt-too-long`, `prompt-blocked-phrase`, `source-invalid`.

### SSE event format

```
event: session-update
data: { "sessionId": "s-4a7b...", "status": "ACTIVE", "turnCount": 3, "lastReply": "..." }
```

One event per entity event (`SessionOpened`, `TurnReplied`, `SessionPaused`, `SessionResumed`, `SessionClosed`, `SessionFailed`). Clients reconcile by `sessionId`; an event always carries the full `SessionRow` at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `submittedBy` from the authenticated principal rather than the request body. Operator-only signals (`pause`, `resume`) should be gated behind a role check in production.
