# API contract — airline-cs

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/requests` | `SubmitServiceRequest` | `201 { requestId }` | `ServiceEndpoint` → `RequestEntity` |
| `GET` | `/api/requests` | — | `200 [ RequestRow... ]` (newest-first) | `ServiceEndpoint` ← `RequestView` |
| `GET` | `/api/requests/{id}` | — | `200 RequestRow` / `404` | `ServiceEndpoint` ← `RequestView` |
| `POST` | `/api/requests/{id}/confirmation` | `ConfirmationAction` | `200 { requestId, status }` | `ServiceEndpoint` → `ServiceWorkflow` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `ServiceEndpoint` ← `RequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitServiceRequest (request body)

```json
{
  "bookingRef": "ABC123",
  "rawMessage": "Hi, I'd like to change my seat on flight AA200 from 22C to something near the window. My email is jane@example.com.",
  "submittedBy": "customer-app-session-77"
}
```

`bookingRef` may be empty for complaint-only flows. `submittedBy` identifies the session; in a production deployment it would be set from the authenticated principal.

### ConfirmationAction (request body for `/api/requests/{id}/confirmation`)

```json
{ "action": "confirm" }
```

or

```json
{ "action": "cancel" }
```

### RequestRow (response body)

```json
{
  "requestId": "req-4c9...",
  "request": {
    "requestId": "req-4c9...",
    "bookingRef": "ABC123",
    "rawMessage": "(raw message preserved for audit)",
    "submittedBy": "customer-app-session-77",
    "receivedAt": "2026-06-28T10:00:00Z"
  },
  "sanitized": {
    "redactedMessage": "Hi, I'd like to change my seat on flight AA200 from 22C to something near the window. My email is [REDACTED-EMAIL].",
    "piiCategoriesFound": ["email"]
  },
  "pendingConfirmation": {
    "proposedChange": "Move from seat 22C to 14A on flight AA200 on 2026-07-10",
    "confirmationToken": "tok_a1b2c3"
  },
  "outcome": {
    "status": "RESOLVED",
    "summary": "Seat changed from 22C to 14A on flight AA200.",
    "toolCallLog": [
      {
        "toolName": "searchBooking",
        "arguments": "{\"bookingRef\":\"ABC123\"}",
        "result": "{\"seat\":\"22C\",\"fareClass\":\"K\",\"upgradeEligible\":true}",
        "calledAt": "2026-06-28T10:00:05Z"
      },
      {
        "toolName": "requestConfirmation",
        "arguments": "{\"proposedChange\":\"Move from seat 22C to 14A on flight AA200 on 2026-07-10\"}",
        "result": "{\"confirmationToken\":\"tok_a1b2c3\"}",
        "calledAt": "2026-06-28T10:00:08Z"
      },
      {
        "toolName": "modifyReservation",
        "arguments": "{\"bookingRef\":\"ABC123\",\"change\":{\"changeType\":\"SEAT\",\"newValue\":\"14A\"},\"confirmationToken\":\"tok_a1b2c3\"}",
        "result": "{\"success\":true,\"newSeat\":\"14A\"}",
        "calledAt": "2026-06-28T10:01:22Z"
      }
    ],
    "caseNumber": null,
    "completedAt": "2026-06-28T10:01:23Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:01:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: request-update
data: { "requestId": "req-4c9...", "status": "AWAITING_CONFIRMATION", "pendingConfirmation": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZED`, `PROCESSING`, `AWAITING_CONFIRMATION`, `COMPLETED`, `CANCELLED`, `FAILED`). Clients reconcile by `requestId`; each event carries the full row at the moment of transition so a late-joining client does not need to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware, set `submittedBy` from the authenticated principal, and validate that the caller of `POST /api/requests/{id}/confirmation` is the same principal who submitted the request.
