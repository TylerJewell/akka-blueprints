# API contract — guardrails-demo

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `SessionEndpoint` → `SessionEntity` |
| `POST` | `/api/sessions/{sessionId}/turns` | `SendTurnRequest` | `201 { turnId }` | `SessionEndpoint` → `SessionWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{sessionId}` | — | `200 SessionRow` / `404` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{sessionId}/sse` | — | `text/event-stream` | `SessionEndpoint` ← `SessionView` |
| `POST` | `/api/sessions/{sessionId}/close` | — | `200 { sessionId }` | `SessionEndpoint` → `SessionEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "userId": "user-42"
}
```

### SendTurnRequest (request body)

```json
{
  "userMessage": "How do I reset my account password?"
}
```

### SessionRow (response body)

```json
{
  "sessionId": "s-9af...",
  "userId": "user-42",
  "turns": [
    {
      "turnId": "t-3bc...",
      "userMessage": "How do I reset my account password?",
      "topicCheck": {
        "outcome": "ALLOWED",
        "matchedTopic": null,
        "reason": "No blocked topic matched."
      },
      "reply": {
        "replyText": "Go to the login page and click 'Forgot password'. Enter your email address and you will receive a reset link within a few minutes.",
        "contentPolicyIterations": 1,
        "repliedAt": "2026-06-28T10:15:34Z"
      },
      "status": "COMPLETED",
      "startedAt": "2026-06-28T10:15:30Z",
      "completedAt": "2026-06-28T10:15:34Z"
    }
  ],
  "status": "OPEN",
  "openedAt": "2026-06-28T10:15:29Z",
  "closedAt": null
}
```

A turn whose topic was blocked:

```json
{
  "turnId": "t-7de...",
  "userMessage": "What stocks should I buy right now?",
  "topicCheck": {
    "outcome": "BLOCKED",
    "matchedTopic": "financial-advice",
    "reason": "Message matched blocked topic: financial-advice"
  },
  "reply": null,
  "status": "BLOCKED",
  "startedAt": "2026-06-28T10:16:00Z",
  "completedAt": "2026-06-28T10:16:00Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-9af...", "status": "OPEN", "turns": [ { ... } ], ... }
```

One event per state transition on any turn within the session (`RECEIVED`, `CHECKING_INPUT`, `BLOCKED`, `GENERATING`, `COMPLETED`, `FAILED`). Clients reconcile by `sessionId` and `turnId`; an event always carries the full session row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `userId` from the authenticated principal rather than the request body.
