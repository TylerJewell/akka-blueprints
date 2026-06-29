# API contract — gemma-food-tour-guide

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tours` | `SubmitTourRequest` | `201 { tourId }` | `TourEndpoint` → `TourRequestEntity` |
| `GET` | `/api/tours` | — | `200 [ TourRequest... ]` (newest-first) | `TourEndpoint` ← `TourView` |
| `GET` | `/api/tours/{id}` | — | `200 TourRequest` / `404` | `TourEndpoint` ← `TourView` |
| `GET` | `/api/tours/sse` | — | `text/event-stream` | `TourEndpoint` ← `TourView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTourRequest (request body)

```json
{
  "city": "Tokyo",
  "durationDays": 3,
  "dietaryStyles": ["vegetarian", "gluten-free"],
  "budgetTier": "STREET_FOOD",
  "notes": "I have celiac disease and prefer smaller neighbourhood spots over tourist traps.",
  "requestedBy": "traveler-88"
}
```

### TourRequest (response body)

```json
{
  "tourId": "t-4ac...",
  "preferences": {
    "city": "Tokyo",
    "durationDays": 3,
    "dietaryStyles": ["vegetarian", "gluten-free"],
    "budgetTier": "STREET_FOOD",
    "notes": "I have celiac disease and prefer smaller neighbourhood spots over tourist traps.",
    "requestedBy": "traveler-88",
    "requestedAt": "2026-06-28T09:00:00Z"
  },
  "validated": {
    "city": "Tokyo",
    "durationDays": 3,
    "normalizedCategories": ["vegetarian", "gluten-free"],
    "budgetTier": "STREET_FOOD"
  },
  "itinerary": {
    "decision": "FULL_PLAN",
    "summary": "Three days of vegetarian, gluten-free street food across Yanaka, Tsukiji, and Shimokitazawa.",
    "days": [
      {
        "dayIndex": 1,
        "stops": [
          {
            "venueId": "tokyo-yanaka-ginza-tofu",
            "venueName": "Yanaka Ginza Tofu Shop",
            "neighborhood": "Yanaka",
            "mealSlot": "BREAKFAST",
            "dietaryCategory": "vegetarian",
            "culturalNote": "Yanaka Ginza is one of Tokyo's few remaining shōtengai, a traditional covered shopping street.",
            "description": "A family-run shop selling fresh silken and firm tofu alongside house-made soy milk. Arrive before 9 am for the freshest blocks; eat standing at the counter."
          }
        ]
      }
    ],
    "generatedAt": "2026-06-28T09:00:22Z"
  },
  "coverage": {
    "score": 4,
    "rationale": "All dietary categories covered across days; day 2 has only two distinct meal slots.",
    "evaluatedAt": "2026-06-28T09:00:23Z"
  },
  "status": "SCORED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: tour-update
data: { "tourId": "t-4ac...", "status": "ITINERARY_RECORDED", "itinerary": { ... }, ... }
```

One event per state transition (`REQUESTED`, `PREFERENCES_VALIDATED`, `GENERATING`, `ITINERARY_RECORDED`, `SCORED`, `FAILED`). Clients reconcile by `tourId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
