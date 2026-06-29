# API contract — airline-triage-router

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/requests` | — | `200 [ FlightRequest... ]` (newest-first); optional `?intent=…&status=…` filtered client-side | `AirlineEndpoint` ← `FlightRequestView` |
| `GET` | `/api/requests/{id}` | — | `200 FlightRequest` / `404` | `AirlineEndpoint` ← `FlightRequestView` |
| `POST` | `/api/requests` | `{ "passengerId": String, "channel": String, "subject": String, "body": String }` | `201 { "requestId": String }` | `AirlineEndpoint` → `PassengerRequestQueue` |
| `POST` | `/api/requests/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 FlightRequest` (status now `RESOLVED`) / `409` if not `BLOCKED` | `AirlineEndpoint` → `FlightRequestEntity` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `AirlineEndpoint` ← `FlightRequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### PassengerRequest (request body for `POST /api/requests`, minus `requestId` and `receivedAt`)

```json
{
  "passengerId": "pax-4422",
  "channel": "web",
  "subject": "My checked bag didn't arrive",
  "body": "Hi, I landed at 14:30 and my bag has not appeared. Bag tag was attached at checkin. Flight was from LHR this morning."
}
```

### FlightRequest

```json
{
  "requestId": "req-7c3f91a2",
  "incoming": {
    "requestId": "req-7c3f91a2",
    "passengerId": "pax-4422",
    "channel": "web",
    "subject": "My checked bag didn't arrive",
    "body": "...raw with PII...",
    "receivedAt": "2026-06-28T14:45:00Z"
  },
  "sanitized": {
    "redactedSubject": "My checked bag didn't arrive",
    "redactedBody": "Hi, I landed at 14:30 and my bag has not appeared. Bag tag was attached at checkin. Flight was from [REDACTED-ROUTE] this morning.",
    "piiCategoriesFound": ["route-info"]
  },
  "intent": {
    "intent": "BAGGAGE",
    "confidence": "high",
    "reason": "Delayed-baggage claim after arrival."
  },
  "resolution": {
    "responseSubject": "Re: My checked bag didn't arrive",
    "responseBody": "We have registered a delayed-baggage claim for your arrival today...",
    "outcome": "BAGGAGE_CLAIM_FILED",
    "specialistTag": "baggage",
    "resolvedAt": "2026-06-28T14:45:22Z"
  },
  "toolCallVerdict": null,
  "responseVerdict": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Intent right; reason names the delayed-arrival signal directly.",
    "scoredAt": "2026-06-28T14:45:14Z"
  },
  "unresolvedReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T14:45:00Z",
  "finishedAt": "2026-06-28T14:45:23Z"
}
```

A blocked ticket (tool-call denial) has `toolCallVerdict.allowed = false`, `toolCallVerdict.denialReason` populated, `status = "BLOCKED"`, `finishedAt = null`.

A blocked ticket (response guardrail) has `responseVerdict.allowed = false`, `responseVerdict.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`.

An unresolved request has `intent.intent = "UNCLEAR"`, `unresolvedReason` populated, `status = "UNRESOLVED"`.

### IntentDecision (returned by `IntentRouter`, embedded in `FlightRequest.intent`)

```json
{ "intent": "BOOKING" | "CHANGE" | "BAGGAGE" | "STATUS" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### AirlineResolution (returned by a specialist, embedded in `FlightRequest.resolution`)

```json
{ "responseSubject": "Re: …",
  "responseBody": "2–5 short paragraphs",
  "outcome": "BOOKING_CONFIRMED" | "CHANGE_APPLIED" | "BAGGAGE_CLAIM_FILED" | "STATUS_PROVIDED" | "FOLLOW_UP_SCHEDULED" | "ESCALATED",
  "specialistTag": "booking" | "change" | "baggage" | "status",
  "resolvedAt": "ISO-8601" }
```

### ToolCallVerdict (returned by `ToolCallGuardrail`, embedded in `FlightRequest.toolCallVerdict`)

```json
{ "allowed": true | false,
  "denialReason": null | "non-refundable-fare" | "outside-change-window" | "name-change-requires-docs" | "authority-limit-exceeded" | "codeshare-partner-action" }
```

### ResponseVerdict (returned by `ResponseGuardrail`, embedded in `FlightRequest.responseVerdict`)

```json
{ "allowed": true | false,
  "violations": ["echoes-pnr-token", "invented-compensation-amount", …],
  "rubricVersion": "v1" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `FlightRequest.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: request-update
data: { "requestId": "req-…", "status": "CLASSIFIED", … full FlightRequest JSON … }
```

One event per state transition on the `FlightRequestView`. Clients reconcile by `requestId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/requests`).
- `404` — unknown `requestId`.
- `409` — `unblock` requested on a request not currently in `BLOCKED`.
- `503` — backend unavailable (workflow timed out, recovery in progress).
