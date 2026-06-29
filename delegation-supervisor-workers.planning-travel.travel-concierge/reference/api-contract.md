# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/trips` | `TripRequest` | `{ "tripId": "uuid" }` | `TripEndpoint` → `TripRequestQueue` |
| GET | `/api/trips` | — | `{ "trips": [TripRow, ...] }` | `TripEndpoint` → `TripView` |
| GET | `/api/trips?status=AWAITING_APPROVAL` | — | filtered list (client-side filter) | `TripEndpoint` |
| GET | `/api/trips/{id}` | — | `TripRow` or 404 | `TripEndpoint` |
| POST | `/api/trips/{id}/confirm` | — | `{ "status": "CONFIRMED" }` or error | `TripEndpoint` → `TripWorkflow` resume |
| POST | `/api/trips/{id}/decline` | — | `{ "status": "DECLINED" }` | `TripEndpoint` → `TripWorkflow` resume |
| GET | `/api/trips/sse` | — | `text/event-stream` of `TripRow` | `TripEndpoint` → `TripView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TripEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TripEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TripEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/trips` request:

```json
{
  "destination": "Lisbon, Portugal",
  "departureDateIso": "2026-09-15",
  "returnDateIso": "2026-09-22",
  "travellerCount": 2,
  "budgetUsd": 3000,
  "requestedBy": "alice@example.com"
}
```

`TripRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "tripId": "uuid",
  "destination": "Lisbon, Portugal",
  "departureDateIso": "2026-09-15",
  "returnDateIso": "2026-09-22",
  "travellerCount": 2,
  "budgetUsd": 3000,
  "status": "PLANNING | RESEARCHING | AWAITING_APPROVAL | CONFIRMED | DECLINED | DEGRADED | BLOCKED",
  "destinationBrief": {
    "destination": "Lisbon, Portugal",
    "highlights": ["string", "..."],
    "advisories": ["string", "..."],
    "visaRequirement": "string",
    "researchedAt": "ISO-8601"
  },
  "fareEstimate": {
    "flightFareUsd": 1200,
    "accommodationFareUsd": 980,
    "totalFareUsd": 2180,
    "currency": "USD",
    "estimatedAt": "ISO-8601"
  },
  "itinerary": {
    "destination": "Lisbon, Portugal",
    "departureDateIso": "2026-09-15",
    "returnDateIso": "2026-09-22",
    "highlights": ["string", "..."],
    "advisories": [],
    "visaRequirement": "string",
    "fare": { "flightFareUsd": 1200, "accommodationFareUsd": 980, "totalFareUsd": 2180, "currency": "USD", "estimatedAt": "ISO-8601" },
    "warnings": null,
    "assembledAt": "ISO-8601"
  },
  "bookingReference": "string or null",
  "failureReason": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/trips/sse` emits one event per trip change:

```
event: trip
data: { ...TripRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `tripId`. No polling. When a row's status transitions to `AWAITING_APPROVAL`, the expanded row renders confirm and decline buttons that call the respective endpoints.
