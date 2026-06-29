# API contract — ui-demo

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `CopilotEndpoint` → `SessionEntity` |
| `POST` | `/api/sessions/{sessionId}/turns` | `SubmitTurnRequest` | `201 { turnId }` | `CopilotEndpoint` → `SessionEntity` + `SessionWorkflow` |
| `GET` | `/api/sessions/{sessionId}/stream` | — | `text/event-stream` (AG-UI envelopes) | `CopilotEndpoint` ← agent task stream |
| `GET` | `/api/sessions/{sessionId}` | — | `200 Session` / `404` | `CopilotEndpoint` ← `SessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `CopilotEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "title": "Event sourcing deep dive"
}
```

### SubmitTurnRequest (request body)

```json
{
  "prompt": "What is event sourcing and why would I use it?"
}
```

### Session (response body — GET /api/sessions/{sessionId})

```json
{
  "sessionId": "s-4a7...",
  "title": "Event sourcing deep dive",
  "turns": [
    {
      "turnId": "t-1b2...",
      "prompt": "What is event sourcing and why would I use it?",
      "response": {
        "answer": "Event sourcing stores the full history of state changes as an immutable sequence of events [1]...",
        "citations": [
          {
            "index": 1,
            "source": "Martin Fowler — Event Sourcing pattern",
            "snippet": "Event Sourcing ensures that all changes to application state are stored as a sequence of events."
          }
        ],
        "suggestedFollowUp": "How do snapshots help manage replay cost in large event journals?",
        "tokenCount": 102,
        "generatedAt": "2026-06-28T12:00:05Z"
      },
      "status": "TURN_COMPLETE",
      "submittedAt": "2026-06-28T12:00:00Z",
      "completedAt": "2026-06-28T12:00:05Z"
    }
  ],
  "status": "IDLE",
  "createdAt": "2026-06-28T11:59:55Z",
  "lastActiveAt": "2026-06-28T12:00:05Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SessionRow (response body — GET /api/sessions list entry)

```json
{
  "sessionId": "s-4a7...",
  "title": "Event sourcing deep dive",
  "status": "IDLE",
  "turnCount": 1,
  "createdAt": "2026-06-28T11:59:55Z",
  "lastActiveAt": "2026-06-28T12:00:05Z"
}
```

## SSE event format (AG-UI protocol)

The stream at `GET /api/sessions/{sessionId}/stream` emits three envelope types:

```
event: TOKEN_CHUNK
data: { "text": "Event sourcing stores" }

event: TOKEN_CHUNK
data: { "text": " the full history" }

event: CITATION_ADDED
data: { "index": 1, "source": "Martin Fowler — Event Sourcing pattern", "snippet": "Event Sourcing ensures..." }

event: RESPONSE_COMPLETE
data: { "turnId": "t-1b2...", "response": { ... full CopilotResponse ... } }
```

- `TOKEN_CHUNK` events are emitted as the agent produces output; the UI appends each to the current chat bubble.
- `CITATION_ADDED` events are emitted when the agent's structured output names a citation; the UI renders the `[n]` marker inline.
- `RESPONSE_COMPLETE` is emitted exactly once per turn, after `TurnCompleted` lands in the entity. It carries the full validated `CopilotResponse`. Clients use this to finalise the bubble and show the suggested-follow-up chip.

The stream closes immediately after `RESPONSE_COMPLETE`. A late-joining client that misses the stream fetches the full session via `GET /api/sessions/{sessionId}` to recover turn history.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap `CopilotEndpoint` with their auth middleware and bind `sessionId` to the authenticated principal so sessions cannot be read across users.
