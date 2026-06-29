# API contract — booking-support-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `SupportEndpoint` → `BookingSessionEntity` |
| `POST` | `/api/sessions/{sessionId}/turns` | `SubmitTurnRequest` | `201 { turnId }` | `SupportEndpoint` → `BookingSessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ BookingSession... ]` (newest-first) | `SupportEndpoint` ← `BookingSessionView` |
| `GET` | `/api/sessions/{sessionId}` | — | `200 BookingSession` / `404` | `SupportEndpoint` ← `BookingSessionView` |
| `GET` | `/api/sessions/{sessionId}/sse` | — | `text/event-stream` | `SupportEndpoint` ← `BookingSessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "customerId": "C001"
}
```

### SubmitTurnRequest (request body)

```json
{
  "customerId": "C001",
  "rawMessage": "What time does my BK-001 flight depart?"
}
```

### BookingSession (response body — full)

```json
{
  "sessionId": "s-4af...",
  "customerId": "C001",
  "turns": [
    {
      "turnId": "t-9cd...",
      "message": {
        "sessionId": "s-4af...",
        "turnId": "t-9cd...",
        "customerId": "C001",
        "rawMessage": "What time does my BK-001 flight depart?",
        "receivedAt": "2026-06-28T14:00:00Z"
      },
      "sanitized": {
        "sanitizedText": "What time does my BK-001 flight depart?",
        "piiCategoriesFound": []
      },
      "reply": {
        "replyText": "Your flight AA101 on booking BK-001 departs at 07:15 UTC on 3 July 2026.",
        "toolCalls": [
          {
            "toolName": "lookUpBooking",
            "targetBookingRef": "BK-001",
            "requestedChange": "look up booking",
            "outcome": "ALLOWED",
            "guardrailReason": null
          }
        ],
        "updatedBooking": null,
        "repliedAt": "2026-06-28T14:00:18Z"
      },
      "status": "AGENT_REPLIED",
      "createdAt": "2026-06-28T14:00:00Z",
      "completedAt": "2026-06-28T14:00:18Z"
    }
  ],
  "sessionStatus": "OPEN",
  "createdAt": "2026-06-28T14:00:00Z",
  "closedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### BookingRecord (embedded in AgentReply.updatedBooking when a mutation occurred)

```json
{
  "bookingRef": "BK-003",
  "customerId": "C001",
  "flightNumber": "BA205",
  "origin": "LHR",
  "destination": "JFK",
  "departureAt": "2026-08-10T10:30Z",
  "arrivalAt": "2026-08-10T13:45Z",
  "seatNumber": "14C",
  "bookingStatus": "CANCELLED",
  "fareClass": "Y",
  "cancellationPermitted": true,
  "seatChangePermitted": true
}
```

### SSE event format

```
event: turn-update
data: { "sessionId": "s-4af...", "turnId": "t-9cd...", "status": "AGENT_REPLIED", "reply": { ... }, ... }
```

One event per turn state transition (`RECEIVED`, `SANITIZED`, `AGENT_REPLIED`, `FAILED`). Clients reconcile by `sessionId` + `turnId`; an event always carries the full turn row at the moment of transition, so a late-joining client does not need to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `customerId` from the authenticated principal rather than the request body.
