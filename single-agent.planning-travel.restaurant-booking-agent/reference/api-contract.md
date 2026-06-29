# API contract — restaurant-booking-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/bookings` | `StartBookingRequest` | `201 { bookingId }` | `BookingEndpoint` → `BookingEntity` + `BookingWorkflow` |
| `GET` | `/api/bookings` | — | `200 [ Booking... ]` (newest-first) | `BookingEndpoint` ← `BookingView` |
| `GET` | `/api/bookings/{id}` | — | `200 Booking` / `404` | `BookingEndpoint` ← `BookingView` |
| `POST` | `/api/bookings/{id}/confirm` | `{}` | `204` | `BookingEndpoint` → `ConfirmationConsumer` |
| `POST` | `/api/bookings/{id}/decline` | `{}` | `204` | `BookingEndpoint` → `ConfirmationConsumer` |
| `GET` | `/api/bookings/sse` | — | `text/event-stream` | `BookingEndpoint` ← `BookingView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartBookingRequest (request body)

```json
{
  "rawRequest": "Book a table for four at Bella Napoli this Friday at 7 pm, for Alex Johnson, email alex@example.com",
  "requestedBy": "user-88421"
}
```

### Booking (response body — CONFIRMED state)

```json
{
  "bookingId": "b-7e3f...",
  "request": {
    "bookingId": "b-7e3f...",
    "rawRequest": "Book a table for four at Bella Napoli this Friday at 7 pm, for Alex Johnson, email alex@example.com",
    "requestedBy": "user-88421",
    "initiatedAt": "2026-07-01T14:00:00Z"
  },
  "proposal": {
    "restaurantId": "bella-napoli",
    "restaurantName": "Bella Napoli",
    "date": "2026-07-04",
    "time": "19:00",
    "partySize": 4,
    "contactName": "Alex Johnson",
    "contactEmail": "alex@example.com",
    "estimatedTotalCents": 10000,
    "proposedAt": "2026-07-01T14:00:15Z"
  },
  "confirmation": {
    "confirmationNumber": "CONF-a3f9c2",
    "confirmedAt": "2026-07-01T14:01:08Z"
  },
  "status": "CONFIRMED",
  "failureReason": null,
  "createdAt": "2026-07-01T14:00:00Z",
  "finishedAt": "2026-07-01T14:01:08Z"
}
```

### Booking (response body — PENDING_CONFIRMATION state)

```json
{
  "bookingId": "b-9a1d...",
  "request": { "..." : "..." },
  "proposal": {
    "restaurantId": "le-petit-bistro",
    "restaurantName": "Le Petit Bistro",
    "date": "2026-07-05",
    "time": "12:30",
    "partySize": 2,
    "contactName": "Maria Chen",
    "contactEmail": "m.chen@example.com",
    "estimatedTotalCents": 6000,
    "proposedAt": "2026-07-01T14:05:30Z"
  },
  "confirmation": null,
  "status": "PENDING_CONFIRMATION",
  "failureReason": null,
  "createdAt": "2026-07-01T14:05:00Z",
  "finishedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: booking-update
data: { "bookingId": "b-7e3f...", "status": "CONFIRMED", "proposal": { ... }, "confirmation": { ... }, ... }
```

One event per state transition (`INITIATED`, `COLLECTING`, `PENDING_CONFIRMATION`, `COMMITTING`, `CONFIRMED`, `DECLINED`, `FAILED`). Clients reconcile by `bookingId`; an event always carries the full row at the moment of transition.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
