# API contract — mcp-github-bot

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `SubmitSessionRequest` | `201 { sessionId }` | `BotSessionEndpoint` → `BotSessionEntity` + `BotSessionWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ BotSessionRow... ]` (newest-first) | `BotSessionEndpoint` ← `BotSessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 BotSessionRow` / `404` | `BotSessionEndpoint` ← `BotSessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `BotSessionEndpoint` ← `BotSessionView` |
| `GET` | `/api/halt` | — | `200 HaltFlagState` | `BotSessionEndpoint` ← `HaltFlagEntity` |
| `POST` | `/api/halt/enable` | `{ operator: String }` | `200 { writeHalted: true }` | `BotSessionEndpoint` → `HaltFlagEntity` |
| `POST` | `/api/halt/disable` | `{ operator: String }` | `200 { writeHalted: false }` | `BotSessionEndpoint` → `HaltFlagEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `BotSessionEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `BotSessionEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `BotSessionEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitSessionRequest (request body)

```json
{
  "userRequest": "List all open issues in acme/widgets",
  "repository": "acme/widgets",
  "githubToken": "ghp_xxxxxxxxxxxx",
  "submittedBy": "developer-7"
}
```

`githubToken` is forwarded to the `McpServerConfig` at agent invocation time. It is NOT stored in `BotSessionEntity` state, NOT included in any SSE event, and NOT returned in any response body.

### BotSessionRow (response body for GET /api/sessions/{id})

```json
{
  "sessionId": "s-4a1...",
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:22Z",
  "response": {
    "agentMessage": "Found 3 open issues in acme/widgets. The highest-priority one is issue #12 'Auth token expiry not handled'.",
    "toolCalls": [
      {
        "toolName": "list_issues",
        "inputSummary": "{\"owner\":\"acme\",\"repo\":\"widgets\",\"state\":\"open\"}",
        "outcome": "success",
        "resultSummary": "[{\"number\":12,\"title\":\"Auth token expiry not handled\"},...] (3 items)",
        "calledAt": "2026-06-28T14:00:01Z"
      }
    ],
    "blockedCallCount": 0,
    "respondedAt": "2026-06-28T14:00:22Z"
  }
}
```

Lifecycle fields that have not been set yet are `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### HaltFlagState (response body for GET /api/halt)

```json
{
  "writeHalted": true,
  "haltedBy": "operator-alice",
  "haltedAt": "2026-06-28T13:55:00Z"
}
```

When the flag has never been set, `haltedBy` and `haltedAt` are `null`.

### SSE event format

```
event: session-update
data: { "sessionId": "s-4a1...", "status": "COMPLETED", "response": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RUNNING`, `COMPLETED`, `FAILED`). Clients reconcile by `sessionId`; every event carries the full row at the moment of transition so a late-joining client never needs to replay history.

## Authorization

ACL: open to the network (local-dev only). A deployer adding identity must wrap `BotSessionEndpoint` with their auth middleware. The `submittedBy` field should be set from the authenticated principal rather than the request body. The halt endpoints should be restricted to operators only — they should not be callable by end users.

The `githubToken` field in `SubmitSessionRequest` is intentionally part of the request body for local development. A production deployer should source the token from a secrets store injected at startup and remove it from the public API surface.
