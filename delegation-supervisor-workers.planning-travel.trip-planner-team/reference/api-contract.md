# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/trips` | `{ destination, preferences, days }` | `{ tripId }` | `TripEndpoint` |
| GET | `/api/trips` | — | `{ trips: [TripPlan, ...] }` | `TripEndpoint` (client-side filter over `TripsView.getAllTrips`) |
| GET | `/api/trips/{tripId}` | — | `TripPlan` \| 404 | `TripEndpoint` |
| GET | `/api/trips/sse` | — | SSE stream of `TripPlan` | `TripEndpoint` |
| GET | `/api/sim/search?q=...` | — | `{ results: [{ title, url, snippet }] }` | `TravelSourceEndpoint` |
| GET | `/api/sim/scrape?url=...` | — | `{ url, body }` \| `{ blocked: true }` | `TravelSourceEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TripEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TripEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TripEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static file | `AppEndpoint` |

## Payload shapes

### POST /api/trips request
```json
{ "destination": "Lisbon", "preferences": "food, walking, slow pace", "days": 3 }
```

### TripPlan (lifecycle fields are nullable — Optional in Java, serialized as raw value or null)
```json
{
  "id": "uuid",
  "destination": "string",
  "preferences": "string",
  "days": 3,
  "status": "GATHERING_INSIGHTS | PLANNING_ITINERARY | PLANNING_LOGISTICS | COMPLETED | FAILED",
  "createdAt": "ISO-8601",
  "localInsights": "string or null",
  "insightsAt": "ISO-8601 or null",
  "itinerary": "string or null",
  "itineraryAt": "ISO-8601 or null",
  "logistics": "string or null",
  "logisticsAt": "ISO-8601 or null",
  "finalPlan": "string or null",
  "completedAt": "ISO-8601 or null",
  "failureReason": "string or null",
  "failedAt": "ISO-8601 or null"
}
```

### Simulated search result
```json
{ "results": [ { "title": "string", "url": "https://...", "snippet": "string" } ] }
```

### Simulated scrape (blocked)
```json
{ "blocked": true, "url": "https://...", "reason": "not allowlisted" }
```

## SSE event format

`GET /api/trips/sse` emits one event per `TripsView` update:

```
event: trip
data: { ...TripPlan... }

```

The UI replaces the matching row by `id` on each event.
