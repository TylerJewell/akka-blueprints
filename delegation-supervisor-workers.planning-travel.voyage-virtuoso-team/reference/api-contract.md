# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/travel` | `{ destination, origin, departureDate, returnDate, travellers, tier }` | `{ "requestId": "uuid" }` | `TravelEndpoint` → `RequestQueue` |
| GET | `/api/travel` | — | `{ "requests": [TravelRequestRow, ...] }` | `TravelEndpoint` → `TravelRequestView` |
| GET | `/api/travel?status=ASSEMBLED` | — | filtered list (client-side filter) | `TravelEndpoint` |
| GET | `/api/travel/{id}` | — | `TravelRequestRow` or 404 | `TravelEndpoint` |
| GET | `/api/travel/sse` | — | `text/event-stream` of `TravelRequestRow` | `TravelEndpoint` → `TravelRequestView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TravelEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TravelEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TravelEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/travel` request:

```json
{
  "destination": "Paris, France",
  "origin": "New York, USA",
  "departureDate": "2026-09-15",
  "returnDate": "2026-09-22",
  "travellers": 2,
  "tier": "business"
}
```

`TravelRequestRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "requestId": "uuid",
  "destination": "Paris, France",
  "origin": "New York, USA",
  "tier": "business",
  "status": "PLANNING | IN_PROGRESS | ASSEMBLED | PARTIAL | BLOCKED",
  "flightPlan": {
    "outboundRouting": "JFK → CDG via AA 100 / nonstop",
    "returnRouting": "CDG → JFK via AA 101 / nonstop",
    "fareClass": "J — business flexible",
    "cabinClass": "business",
    "notes": ["Flat-bed seat", "Two checked bags included", "Lounge access"],
    "gatheredAt": "ISO-8601"
  },
  "lodgingPlan": {
    "propertyName": "Le Bristol Paris",
    "propertyCategory": "urban luxury hotel",
    "roomType": "Deluxe King",
    "highlights": ["8th arrondissement", "Michelin-starred restaurant on-site"],
    "gatheredAt": "ISO-8601"
  },
  "experiencePlan": {
    "activities": ["Private guided Louvre tour (3 h)", "Seine river cruise (1.5 h)"],
    "dining": ["Bistrot Paul Bert — classic Parisian bistro", "Septime — contemporary French, book ahead"],
    "cultural": ["Musée d'Orsay — Impressionist collection", "Le Marais — historic neighbourhood walk"],
    "gatheredAt": "ISO-8601"
  },
  "logisticsPlan": {
    "groundTransfers": ["CDG to hotel: private chauffeur (~45 min)", "RER B to Gare du Nord (~35 min, economy alternative)"],
    "entryRequirements": ["US passport holders: no visa required for stays up to 90 days (as of general knowledge)", "Passport valid 6+ months beyond return date"],
    "travelAdvisories": ["Standard urban precautions in tourist areas", "Verify current FCO / State Department advisory before travel"],
    "gatheredAt": "ISO-8601"
  },
  "itinerary": {
    "synopsis": "string (100–160 words)",
    "guardrailVerdict": "ok",
    "assembledAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/travel/sse` emits one event per request change:

```
event: travel-request
data: { ...TravelRequestRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `requestId`. No polling.
