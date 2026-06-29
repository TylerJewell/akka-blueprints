# API contract — ae-oauth

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `SessionEndpoint` → `SessionEntity` + `SessionWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (newest-first) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "tokenId": "read-only",
  "requestText": "List my calendar events and create a new one for next Friday at 2 PM.",
  "submittedBy": "app-user-7"
}
```

### Session (response body)

```json
{
  "sessionId": "s-9fc...",
  "request": {
    "sessionId": "s-9fc...",
    "tokenId": "read-only",
    "requestText": "List my calendar events and create a new one for next Friday at 2 PM.",
    "submittedBy": "app-user-7",
    "submittedAt": "2026-06-28T14:10:00Z"
  },
  "token": {
    "tokenId": "read-only",
    "scopes": ["calendar:read", "files:read", "contacts:read"],
    "expired": false
  },
  "result": {
    "outcome": "PARTIAL",
    "summary": "Calendar events listed successfully. The new event could not be created: token lacks calendar:write scope.",
    "toolCalls": [
      {
        "toolName": "listCalendarEvents",
        "requiredScope": "calendar:read",
        "disposition": "ALLOWED",
        "output": "[{\"id\":\"evt-1\",\"title\":\"Team sync\",\"start\":\"2026-07-01T09:00:00Z\"},{\"id\":\"evt-2\",\"title\":\"Product review\",\"start\":\"2026-07-02T14:00:00Z\"},{\"id\":\"evt-3\",\"title\":\"1:1 with manager\",\"start\":\"2026-07-03T11:00:00Z\"}]",
        "calledAt": "2026-06-28T14:10:08Z"
      },
      {
        "toolName": "createCalendarEvent",
        "requiredScope": "calendar:write",
        "disposition": "DENIED",
        "output": "token lacks scope calendar:write",
        "calledAt": "2026-06-28T14:10:09Z"
      }
    ],
    "completedAt": "2026-06-28T14:10:10Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:10:00Z",
  "finishedAt": "2026-06-28T14:10:10Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-9fc...", "status": "COMPLETED", "result": { ... }, ... }
```

One event per state transition (`PENDING`, `TOKEN_RESOLVED`, `RUNNING`, `COMPLETED`, `FAILED`). Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer integrating a real OAuth provider must validate the inbound token against the authorization server, resolve the actual scope set from the token's introspection response, and pass only the confirmed scope list into `SessionWorkflow.resolveTokenStep`. The `TokenRegistry` in this blueprint simulates that step in-process.
