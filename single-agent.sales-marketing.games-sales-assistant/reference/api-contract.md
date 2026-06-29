# API contract — games-sales-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `SessionEndpoint` → `SessionEntity` |
| `POST` | `/api/sessions/{sessionId}/turns` | `PostTurnRequest` | `201 { turnId }` | `SessionEndpoint` → `SessionWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SessionEndpoint` ← `SessionEntity` |
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
  "shopperId": "shopper-9182"
}
```

### PostTurnRequest (request body)

```json
{
  "question": "What action RPGs do you have under $40 on PC?"
}
```

### Session (response body — full)

```json
{
  "sessionId": "sess-4f7...",
  "shopperId": "shopper-9182",
  "turns": [
    {
      "turnId": "turn-001",
      "sessionId": "sess-4f7...",
      "request": {
        "turnId": "turn-001",
        "question": "What action RPGs do you have under $40 on PC?",
        "context": {
          "sessionId": "sess-4f7...",
          "shopperId": "shopper-9182",
          "recentOrders": [],
          "previousTurnIds": []
        },
        "askedAt": "2026-06-28T14:22:00Z"
      },
      "response": {
        "answer": "We have two action RPGs under $40 on PC in stock right now.",
        "recommendations": [
          {
            "titleId": "steel-hollow-pc",
            "name": "Steel Hollow",
            "platform": "PC",
            "priceUsd": 29.99,
            "pitch": "An open-world action RPG with deep crafting and a 40-hour main story."
          },
          {
            "titleId": "verdant-siege-pc",
            "name": "Verdant Siege",
            "platform": "PC",
            "priceUsd": 34.99,
            "pitch": "A tactical action RPG set in a post-collapse fantasy world with branching faction choices."
          }
        ],
        "orders": [],
        "status": "ANSWERED",
        "answeredAt": "2026-06-28T14:22:18Z"
      },
      "status": "ANSWERED",
      "createdAt": "2026-06-28T14:22:00Z",
      "answeredAt": "2026-06-28T14:22:18Z"
    }
  ],
  "status": "ACTIVE",
  "createdAt": "2026-06-28T14:21:55Z",
  "closedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SessionRow (list response entry)

```json
{
  "sessionId": "sess-4f7...",
  "shopperId": "shopper-9182",
  "turnCount": 1,
  "lastTurnStatus": "ANSWERED",
  "lastTurnAnsweredAt": "2026-06-28T14:22:18Z",
  "sessionStatus": "ACTIVE",
  "createdAt": "2026-06-28T14:21:55Z"
}
```

### SSE event format

```
event: session-update
data: { "sessionId": "sess-4f7...", "sessionStatus": "ACTIVE", "lastTurnStatus": "ANSWERED", "turnCount": 1, ... }
```

One event per state transition (`TurnStarted`, `TurnAnswered`, `TurnRejected`, `TurnFailed`, `SessionClosed`, `SessionFailed`). Clients reconcile by `sessionId`; an event always carries the full `SessionRow` at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `shopperId` from the authenticated principal rather than the request body.
