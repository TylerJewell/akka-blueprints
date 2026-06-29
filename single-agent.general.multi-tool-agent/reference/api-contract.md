# API contract — multi-tool-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `SubmitSessionRequest` | `201 { sessionId }` | `SessionEndpoint` → `SessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (newest-first) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitSessionRequest (request body)

```json
{
  "requestText": "What is 72°F in Celsius, and how many euros is $85?",
  "submittedBy": "user-42"
}
```

### Session (response body)

```json
{
  "sessionId": "s-9af...",
  "request": {
    "sessionId": "s-9af...",
    "requestText": "What is 72°F in Celsius, and how many euros is $85?",
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "toolCalls": [
    {
      "callId": "c-001",
      "toolName": "UnitTool",
      "inputArgs": {
        "value": "72.0",
        "fromUnit": "fahrenheit",
        "toUnit": "celsius"
      },
      "result": "{\"fromUnit\":\"fahrenheit\",\"toUnit\":\"celsius\",\"inputValue\":72.0,\"outputValue\":22.22}",
      "guardRejectionReason": null,
      "calledAt": "2026-06-28T10:00:05Z"
    },
    {
      "callId": "c-002",
      "toolName": "CurrencyTool",
      "inputArgs": {
        "amount": "85.0",
        "fromCode": "USD",
        "toCode": "EUR"
      },
      "result": "{\"fromCode\":\"USD\",\"toCode\":\"EUR\",\"amount\":85.0,\"converted\":78.20,\"rate\":0.92}",
      "guardRejectionReason": null,
      "calledAt": "2026-06-28T10:00:07Z"
    }
  ],
  "response": {
    "answer": "72°F is approximately 22.2°C. $85 converts to approximately €78.20 at the current rate.",
    "toolCalls": [ "... (same list as above)" ],
    "answeredAt": "2026-06-28T10:00:10Z"
  },
  "status": "ANSWERED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:10Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-9af...", "status": "ANSWERED", "response": { ... }, "toolCalls": [ ... ] }
```

One event per state transition (`SUBMITTED`, `DISPATCHING`, `ANSWERED`, `FAILED`). Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
