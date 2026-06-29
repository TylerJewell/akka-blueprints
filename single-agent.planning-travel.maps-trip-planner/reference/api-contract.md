# API contract — maps-trip-planner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/trips` | `SubmitTripRequest` | `201 { tripId }` | `PlanningEndpoint` → `TripRequestEntity` |
| `GET` | `/api/trips` | — | `200 [ TripRow... ]` (newest-first) | `PlanningEndpoint` ← `ItineraryView` |
| `GET` | `/api/trips/{id}` | — | `200 TripRow` / `404` | `PlanningEndpoint` ← `ItineraryView` |
| `GET` | `/api/trips/sse` | — | `text/event-stream` | `PlanningEndpoint` ← `ItineraryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTripRequest (request body)

```json
{
  "originCity": "London",
  "destinations": ["Paris"],
  "startDate": "2026-09-10",
  "endDate": "2026-09-12",
  "partySize": 2,
  "preferencesText": "vegetarian meals, cultural landmarks, avoid crowded tourist traps",
  "requestedBy": "traveller-42"
}
```

### TripRow (response body — full example at EVALUATED)

```json
{
  "tripId": "t-7a3...",
  "request": {
    "tripId": "t-7a3...",
    "originCity": "London",
    "destinations": ["Paris"],
    "startDate": "2026-09-10",
    "endDate": "2026-09-12",
    "partySize": 2,
    "preferencesText": "vegetarian meals, cultural landmarks, avoid crowded tourist traps",
    "requestedBy": "traveller-42",
    "submittedAt": "2026-09-01T08:00:00Z"
  },
  "screen": {
    "decision": "PASS",
    "reason": "ok"
  },
  "itinerary": {
    "quality": "EXCELLENT",
    "summary": "Three days in Paris balancing cultural landmarks with vegetarian-friendly dining.",
    "days": [
      {
        "dayNumber": 1,
        "date": "2026-09-10",
        "stops": [
          {
            "placeName": "Musée d'Orsay",
            "geocodedAddress": "1 Rue de la Légion d'Honneur, 75007 Paris",
            "latDeg": 48.8600,
            "lngDeg": 2.3266,
            "visitMinutes": 120,
            "description": "Impressionist and post-impressionist art in a former railway station."
          },
          {
            "placeName": "Le Potager de Charlotte",
            "geocodedAddress": "12 Rue de la Vrillière, 75001 Paris",
            "latDeg": 48.8635,
            "lngDeg": 2.3406,
            "visitMinutes": 75,
            "description": "Vegetarian restaurant in the 1st arrondissement; seasonal French menu."
          }
        ],
        "legs": [
          {
            "fromPlace": "Musée d'Orsay",
            "toPlace": "Le Potager de Charlotte",
            "mode": "TRANSIT",
            "durationMinutes": 18
          }
        ],
        "dayNarrative": "Morning immersed in Impressionism at the Orsay, afternoon transit to a vegetarian lunch in the 1st."
      }
    ],
    "preferencesAddressed": ["vegetarian meals", "cultural landmarks"],
    "decidedAt": "2026-09-01T08:00:28Z"
  },
  "eval": {
    "score": 5,
    "rationale": "All preference keywords addressed; per-day activity windows are plausible; all transit legs connect consecutive stops.",
    "evaluatedAt": "2026-09-01T08:00:29Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-09-01T08:00:00Z",
  "finishedAt": "2026-09-01T08:00:29Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### Blocked trip example

```json
{
  "tripId": "t-9b1...",
  "request": { ... },
  "screen": {
    "decision": "BLOCKED",
    "reason": "Destination matches blocked keyword: demo-block-a"
  },
  "itinerary": null,
  "eval": null,
  "status": "BLOCKED",
  "createdAt": "2026-09-01T09:00:00Z",
  "finishedAt": "2026-09-01T09:00:01Z"
}
```

### SSE event format

```
event: trip-update
data: { "tripId": "t-7a3...", "status": "ITINERARY_RECORDED", "itinerary": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SCREENED`, `BLOCKED`, `PLANNING`, `ITINERARY_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `tripId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
