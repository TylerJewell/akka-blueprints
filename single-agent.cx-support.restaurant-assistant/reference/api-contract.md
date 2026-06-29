# API contract — restaurant-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `SessionEndpoint` → `SessionEntity` |
| `POST` | `/api/sessions/{id}/messages` | `SendMessageRequest` | `202 { messageId, sessionId }` | `SessionEndpoint` → `SessionWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 SessionRow` / `404` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `SessionEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `SessionEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `SessionEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "guestName": "Alex Patel",
  "contactPhone": "07700900123"
}
```

Both fields are optional. If omitted, the agent collects them during the conversation before committing a reservation.

### SendMessageRequest (request body)

```json
{
  "text": "I'd like to book a table for 2 on Friday evening."
}
```

### SessionRow (response body — full example after a reservation)

```json
{
  "sessionId": "s-9ae...",
  "messages": [
    {
      "messageId": "m-001",
      "role": "CUSTOMER",
      "text": "I'd like to book a table for 2 on Friday evening.",
      "sentAt": "2026-07-01T18:44:00Z"
    },
    {
      "messageId": "m-002",
      "role": "ASSISTANT",
      "text": "Happy to help — which Friday did you have in mind, and what time works for you?",
      "sentAt": "2026-07-01T18:44:06Z"
    },
    {
      "messageId": "m-003",
      "role": "CUSTOMER",
      "text": "Friday 4th at 7pm.",
      "sentAt": "2026-07-01T18:44:30Z"
    },
    {
      "messageId": "m-004",
      "role": "ASSISTANT",
      "text": "I'll book a table for 2 on Friday 4 July at 19:00 for Alex Patel. Shall I confirm?",
      "sentAt": "2026-07-01T18:44:37Z"
    },
    {
      "messageId": "m-005",
      "role": "CUSTOMER",
      "text": "Yes please.",
      "sentAt": "2026-07-01T18:44:45Z"
    },
    {
      "messageId": "m-006",
      "role": "ASSISTANT",
      "text": "Done — table booked for 2 on Friday 4 July at 19:00. See you then!",
      "sentAt": "2026-07-01T18:44:52Z"
    }
  ],
  "reservation": {
    "reservationId": "rsv-7f1...",
    "details": {
      "date": "2026-07-04",
      "time": "19:00",
      "partySize": 2,
      "guestName": "Alex Patel",
      "contactPhone": "07700900123"
    },
    "confirmedAt": "2026-07-01T18:44:52Z"
  },
  "order": null,
  "status": "RESERVATION_HELD",
  "openedAt": "2026-07-01T18:44:00Z",
  "closedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-9ae...", "status": "RESERVATION_HELD", "reservation": { ... }, ... }
```

One event per state transition (`OPEN`, `ACTIVE`, `RESERVATION_HELD`, `ORDER_PLACED`, `CLOSED`, `FAILED`). Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay history.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `guestName` / `contactPhone` from the authenticated session rather than the request body.
