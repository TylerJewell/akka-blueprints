# API contract — realtime-conversational-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `StartSessionRequest` | `201 { sessionId }` | `SessionEndpoint` → `SessionEntity`, `SessionWorkflow` |
| `POST` | `/api/sessions/{id}/turns` | `SubmitTurnRequest` | `202 { turnId }` | `SessionEndpoint` → `SessionWorkflow` |
| `POST` | `/api/sessions/{id}/end` | — | `204` | `SessionEndpoint` → `SessionWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (newest-first) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body)

```json
{
  "customerId": "customer-8821",
  "agentPersona": "retail-support"
}
```

`agentPersona` is optional; defaults to `"general-support"` if omitted.

### SubmitTurnRequest (request body)

```json
{
  "text": "I'd like to check the status of my order from last Tuesday."
}
```

### Session (response body)

```json
{
  "sessionId": "s-4f9a...",
  "customerId": "customer-8821",
  "agentPersona": "retail-support",
  "turns": [
    {
      "turnId": "t-001",
      "customer": {
        "turnId": "t-001",
        "text": "<<GREETING>>",
        "receivedAt": "2026-06-28T09:00:00Z"
      },
      "agent": {
        "turnId": "t-001",
        "replyText": "Hello! I'm here to help with your order. What can I do for you today?",
        "guardrailTriggered": false,
        "iterationsUsed": 1,
        "repliedAt": "2026-06-28T09:00:03Z"
      },
      "status": "AGENT_REPLIED",
      "startedAt": "2026-06-28T09:00:00Z"
    },
    {
      "turnId": "t-002",
      "customer": {
        "turnId": "t-002",
        "text": "I'd like to check the status of my order from last Tuesday.",
        "receivedAt": "2026-06-28T09:00:15Z"
      },
      "agent": {
        "turnId": "t-002",
        "replyText": "I can help with that. Could you share your order number so I can look that up?",
        "guardrailTriggered": false,
        "iterationsUsed": 1,
        "repliedAt": "2026-06-28T09:00:21Z"
      },
      "status": "AGENT_REPLIED",
      "startedAt": "2026-06-28T09:00:15Z"
    }
  ],
  "summary": {
    "qualityScore": 5,
    "summaryText": "Two-turn session; customer asked about an order; agent requested the order number. No guardrail events.",
    "totalTurns": 2,
    "guardrailEvents": 0,
    "summarizedAt": "2026-06-28T09:01:00Z"
  },
  "status": "CLOSED",
  "openedAt": "2026-06-28T09:00:00Z",
  "closedAt": "2026-06-28T09:01:00Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). While a turn is in progress, `agent` is `null` and `status` is `WAITING_FOR_AGENT`.

### SSE event format

```
event: session-update
data: { "sessionId": "s-4f9a...", "status": "ACTIVE", "turns": [ ... ], ... }
```

One event per state transition (`GREETING`, `ACTIVE`, `WAITING_FOR_AGENT`, `SUMMARIZING`, `CLOSED`, `FAILED`) and per completed turn (one event when a turn's `AgentReplied` event lands). Clients reconcile by `sessionId`; an event always carries the full session row at the moment of the transition, so a late-joining client sees the complete current state without replaying entity events.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `customerId` from the authenticated principal rather than the request body.
