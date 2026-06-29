# API contract — travel-support-router

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/requests` | — | `200 [ TravelRequest... ]` (newest-first); supports optional `?category=…&status=…` filtered client-side | `TravelEndpoint` ← `RequestView` |
| `GET` | `/api/requests/{id}` | — | `200 TravelRequest` / `404` | `TravelEndpoint` ← `RequestView` |
| `POST` | `/api/requests` | `{ "passengerId": String, "channel": String, "subject": String, "body": String }` | `201 { "requestId": String }` | `TravelEndpoint` → `RequestQueue` |
| `POST` | `/api/requests/{id}/confirm` | `{ "outcome": "CONFIRM" \| "REJECT", "passengerId": String }` | `200 TravelRequest` (status updated) / `409` if not `AWAITING_CONFIRMATION` | `TravelEndpoint` → `RequestEntity` |
| `POST` | `/api/requests/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 TravelRequest` (status now `RESOLVED`) / `409` if not `BLOCKED` | `TravelEndpoint` → `RequestEntity` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `TravelEndpoint` ← `RequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingRequest (request body for `POST /api/requests`, minus `requestId` and `receivedAt`)

```json
{
  "passengerId": "pax-4821",
  "channel": "web-chat",
  "subject": "Change my seat on tomorrow's flight",
  "body": "Hi, I need to move to an aisle seat on my morning departure. Booking ref SK774201. Passport: AB1234567."
}
```

### TravelRequest

```json
{
  "requestId": "req-a3f91c4b",
  "incoming": {
    "requestId": "req-a3f91c4b",
    "passengerId": "pax-4821",
    "channel": "web-chat",
    "subject": "Change my seat on tomorrow's flight",
    "body": "...raw with PII...",
    "receivedAt": "2026-06-28T08:10:00Z"
  },
  "sanitized": {
    "redactedSubject": "Change my seat on tomorrow's flight",
    "redactedBody": "Hi, I need to move to an aisle seat on my morning departure. Booking ref [REDACTED-BOOKING-REF]. Passport: [REDACTED-PASSPORT].",
    "piiCategoriesFound": ["booking-reference", "passport-number"]
  },
  "routing": {
    "category": "FLIGHTS",
    "confidence": "high",
    "reason": "Explicit seat-change request on a flight segment."
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "confirmation": {
    "summary": "Your booking (ref: SK774201) seat will be changed to an aisle seat — please confirm to proceed.",
    "deadline": "2026-06-28T08:20:00Z"
  },
  "confirmationOutcome": "CONFIRM",
  "resolution": {
    "responseSubject": "Re: Change my seat on tomorrow's flight",
    "responseBody": "Your seat has been changed to an aisle seat...",
    "action": "BOOKING_CHANGED",
    "specialistTag": "flights",
    "bookingRef": "SK774201",
    "resolvedAt": "2026-06-28T08:10:22Z"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Category right; reason names the seat-change signal directly.",
    "scoredAt": "2026-06-28T08:10:14Z"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T08:10:00Z",
  "finishedAt": "2026-06-28T08:10:25Z"
}
```

A blocked request has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. A passenger-rejected request has `confirmationOutcome = "REJECT"`, `status = "PASSENGER_REJECTED"`, `finishedAt` set, no mutation applied. An escalated request has `routing.category = "UNCLEAR"`, `escalationReason` populated, `status = "ESCALATED"`.

### RoutingDecision (returned by `RouterAgent`, embedded in `TravelRequest.routing`)

```json
{ "category": "FLIGHTS" | "HOTELS" | "CAR_RENTAL" | "EXCURSIONS" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### Resolution (returned by a specialist, embedded in `TravelRequest.resolution`)

```json
{ "responseSubject": "Re: …",
  "responseBody": "3–5 short paragraphs",
  "action": "BOOKING_CHANGED" | "BOOKING_CANCELLED" | "BOOKING_CONFIRMED" | "INFO_PROVIDED" | "ARTICLE_LINKED" | "FOLLOW_UP_SCHEDULED" | "ESCALATED",
  "specialistTag": "flights" | "hotels" | "car-rental" | "excursions",
  "bookingRef": "SK774201" | null,
  "resolvedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `BookingGuardrail`, embedded in `TravelRequest.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["non-refundable-outside-window", "upgrade-above-entitlement", …],
  "rubricVersion": "v1" }
```

### ConfirmationRequest (returned by `ConfirmationGate`, embedded in `TravelRequest.confirmation`)

```json
{ "summary": "Your booking (ref: SK774201) seat will be changed to an aisle seat — please confirm to proceed.",
  "deadline": "ISO-8601" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `TravelRequest.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

### POST /api/requests/{id}/confirm body

```json
{ "outcome": "CONFIRM" | "REJECT",
  "passengerId": "pax-4821" }
```

### POST /api/requests/{id}/unblock body

```json
{ "decidedBy": "supervisor-handle",
  "note": "Verified with flight-ops; non-refundable exception approved." }
```

## SSE event format

```
event: request-update
data: { "requestId": "req-…", "status": "ROUTED_FLIGHTS", … full TravelRequest JSON … }
```

One event per state transition on the `RequestView`. Clients reconcile by `requestId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/requests`).
- `404` — unknown `requestId`.
- `409` — `confirm` requested on a request not currently `AWAITING_CONFIRMATION`; or `unblock` requested on a request not currently `BLOCKED`.
- `503` — backend unavailable (workflow timed out, recovery in progress).
