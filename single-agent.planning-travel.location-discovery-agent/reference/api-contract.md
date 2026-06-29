# API contract — location-discovery-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/searches` | `SubmitSearchRequest` | `201 { searchId }` | `SearchEndpoint` → `SearchEntity` |
| `GET` | `/api/searches` | — | `200 [ Search... ]` (newest-first) | `SearchEndpoint` ← `SearchView` |
| `GET` | `/api/searches/{id}` | — | `200 Search` / `404` | `SearchEndpoint` ← `SearchView` |
| `GET` | `/api/searches/sse` | — | `text/event-stream` | `SearchEndpoint` ← `SearchView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitSearchRequest (request body)

```json
{
  "query": "coffee shops with good wifi for remote work",
  "rawLatitude": 37.7831,
  "rawLongitude": -122.4159,
  "radiusMeters": 1000,
  "categoryFilter": "Food & Drink",
  "submittedBy": "traveler-session-42"
}
```

`categoryFilter` is optional; omit or set to `null` to search all categories.

### Search (response body)

```json
{
  "searchId": "s-8c3...",
  "request": {
    "searchId": "s-8c3...",
    "query": "coffee shops with good wifi for remote work",
    "rawLatitude": 37.7831,
    "rawLongitude": -122.4159,
    "radiusMeters": 1000,
    "categoryFilter": "Food & Drink",
    "submittedBy": "traveler-session-42",
    "submittedAt": "2026-06-28T15:20:00Z"
  },
  "sanitized": {
    "boundingBoxToken": "bbox:37.77-37.79:-122.42--122.40",
    "candidatePlaces": [
      {
        "placeId": "fsq-sf-001",
        "name": "Ritual Coffee Roasters",
        "category": "Coffee Shop",
        "distanceMeters": 320.0,
        "latitude": 37.7812,
        "longitude": -122.4142,
        "address": "1026 Valencia St",
        "rating": 4.7,
        "reviewCount": 1840
      }
    ]
  },
  "recommendation": {
    "summary": "Two cafes match well for remote work; the third is a quick-service counter less suited to extended stays.",
    "places": [
      {
        "placeId": "fsq-sf-001",
        "name": "Ritual Coffee Roasters",
        "category": "Coffee Shop",
        "distanceMeters": 320.0,
        "relevanceScore": 9,
        "rationale": "Specialty coffee roaster 320 m away with ample seating and a reputation for reliable wifi among local remote workers."
      }
    ],
    "decidedAt": "2026-06-28T15:20:22Z"
  },
  "eval": {
    "score": 4,
    "rationale": "All rationales are specific and distance-aware; one result could have a broader score spread.",
    "evaluatedAt": "2026-06-28T15:20:23Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T15:20:00Z",
  "finishedAt": "2026-06-28T15:20:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: search-update
data: { "searchId": "s-8c3...", "status": "RECOMMENDATION_RECORDED", "recommendation": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `COORDINATE_SANITIZED`, `DISCOVERING`, `RECOMMENDATION_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `searchId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body. Raw coordinates in the request body must be treated as sensitive; HTTPS is required in any non-local deployment.
