# API contract — flight-booking-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/bookings` | `SearchRequest` | `201 { bookingId }` | `BookingEndpoint` → `BookingEntity` |
| `POST` | `/api/bookings/{id}/confirm` | — | `200 { bookingId, status }` | `BookingEndpoint` → `BookingEntity` |
| `GET` | `/api/bookings` | — | `200 [ BookingRecord... ]` (newest-first) | `BookingEndpoint` ← `BookingView` |
| `GET` | `/api/bookings/{id}` | — | `200 BookingRecord` / `404` | `BookingEndpoint` ← `BookingView` |
| `GET` | `/api/bookings/sse` | — | `text/event-stream` | `BookingEndpoint` ← `BookingView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SearchRequest (request body for POST /api/bookings)

```json
{
  "origin": "SFO",
  "destination": "JFK",
  "travelDate": "2026-09-15"
}
```

### BookingRecord (response body)

```json
{
  "bookingId": "bk-7d3e9a...",
  "origin": "SFO",
  "destination": "JFK",
  "travelDate": "2026-09-15",
  "flightOffers": {
    "offers": [
      {
        "flightId": "fl-aa100-sfo-jfk-20260915",
        "carrier": "American Airlines",
        "flightNumber": "AA100",
        "origin": "SFO",
        "destination": "JFK",
        "departureAt": "2026-09-15T08:00:00Z",
        "arrivalAt": "2026-09-15T16:30:00Z",
        "cabinClass": "ECONOMY",
        "fareUsdCents": 32500
      }
    ],
    "searchedAt": "2026-06-28T10:00:00Z"
  },
  "seatSelection": {
    "flightId": "fl-aa100-sfo-jfk-20260915",
    "carrier": "American Airlines",
    "flightNumber": "AA100",
    "origin": "SFO",
    "destination": "JFK",
    "departureAt": "2026-09-15T08:00:00Z",
    "arrivalAt": "2026-09-15T16:30:00Z",
    "seatCode": "22A",
    "cabinClass": "ECONOMY",
    "totalCostUsdCents": 32500,
    "selectedAt": "2026-06-28T10:00:10Z"
  },
  "confirmation": {
    "confirmationCode": "AAXQ99",
    "passengerRef": "PAX-001",
    "flightId": "fl-aa100-sfo-jfk-20260915",
    "carrier": "American Airlines",
    "flightNumber": "AA100",
    "origin": "SFO",
    "destination": "JFK",
    "departureAt": "2026-09-15T08:00:00Z",
    "seatCode": "22A",
    "cabinClass": "ECONOMY",
    "totalCostUsdCents": 32500,
    "committedAt": "2026-06-28T10:00:30Z"
  },
  "status": "COMMITTED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:30Z",
  "guardrailRejections": []
}
```

A booking that has just been submitted but not yet searched will have `flightOffers`, `seatSelection`, and `confirmation` as `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). A booking in `AWAITING_CONFIRMATION` will have `flightOffers` and `seatSelection` present but `confirmation` null.

An example with a guardrail rejection:

```json
{
  "bookingId": "bk-3c1f7b...",
  "status": "COMMITTED",
  "guardrailRejections": [
    {
      "phase": "SEARCH",
      "tool": "getSeatMap",
      "reason": "phase-violation: getSeatMap requires status in {SEARCH_DONE, SELECTING} with flightOffers present, saw SEARCHING",
      "rejectedAt": "2026-06-28T10:00:01Z"
    }
  ]
}
```

### Confirm response body

```json
{
  "bookingId": "bk-7d3e9a...",
  "status": "CONFIRMED"
}
```

### SSE event format

```
event: booking-update
data: { "bookingId": "bk-7d3e9a...", "status": "AWAITING_CONFIRMATION", "seatSelection": { ... }, ... }
```

One event per state transition (`CREATED`, `SEARCHING`, `SEARCH_DONE`, `SELECTING`, `SEAT_SELECTED`, `AWAITING_CONFIRMATION`, `CONFIRMED`, `BOOKING`, `COMMITTED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: booking-rejection
data: { "bookingId": "bk-7d3e9a...", "phase": "SEARCH", "tool": "getSeatMap", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `bookingId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SearchRequest` record and the `BookingCreated` event to carry it. The `/confirm` endpoint should additionally verify that the confirming user is the same principal who submitted the booking.
