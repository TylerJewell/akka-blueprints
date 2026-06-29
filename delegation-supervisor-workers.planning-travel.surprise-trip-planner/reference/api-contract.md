# API contract — surprise-trip-planner

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/trips` | `PreferenceSet` | `{ "tripId": "uuid" }` | TripEndpoint → InboundRequestQueue |
| GET | `/api/trips?status=...` | — | `{ "trips": [Trip, ...] }` | TripEndpoint → TripsView (client-side filter) |
| GET | `/api/trips/{tripId}` | — | `Trip` | TripEndpoint → TripsView |
| GET | `/api/trips/sse` | — | SSE stream of `Trip` | TripEndpoint → TripsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | TripEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | TripEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | TripEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

The `status` filter is applied in the endpoint after `getAllTrips`, because the view cannot auto-index the enum column (Lesson 2).

## Payload shapes

`PreferenceSet` (request body):

```json
{
  "vibe": "string",
  "budgetBand": "low | mid | high",
  "startDate": "YYYY-MM-DD",
  "endDate": "YYYY-MM-DD",
  "partySize": 2,
  "noGo": ["string", "..."]
}
```

`Trip` (response; lifecycle fields are `Optional` in Java, serialized as raw value or `null`):

```json
{
  "id": "uuid",
  "preferences": { "...PreferenceSet..." },
  "status": "REQUESTED | RESEARCHED | READY | ESCALATED | FAILED",
  "requestedAt": "ISO-8601 or null",
  "destinationName": "string or null",
  "destinationRationale": "string or null",
  "researchedAt": "ISO-8601 or null",
  "logistics": { "transport": "string", "lodging": "string", "estCostBand": "string" },
  "activities": [ { "day": 1, "summary": "string", "items": ["string"] } ],
  "assembledAt": "ISO-8601 or null",
  "groundingNotes": "string or null",
  "blockedToolNote": "string or null",
  "escalatedAt": "ISO-8601 or null",
  "failureReason": "string or null"
}
```

Before status is `READY`, `destinationName` and `destinationRationale` are withheld (null) so the surprise holds.

## SSE format

`GET /api/trips/sse` emits one `data:` line per `Trip` update, JSON-encoded, terminated by a blank line. The UI subscribes with `EventSource` and upserts each trip into the live list by `id`.
