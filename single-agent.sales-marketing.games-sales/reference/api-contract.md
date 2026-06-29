# API contract — games-sales

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `SalesEndpoint` → `SessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (newest-first) | `SalesEndpoint` ← `RecommendationView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SalesEndpoint` ← `RecommendationView` |
| `POST` | `/api/sessions/{id}/follow-up` | `FollowUpRequest` | `200 { sessionId }` | `SalesEndpoint` → `SessionEntity` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `SalesEndpoint` ← `RecommendationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "query": "I want a co-op action game for two players on PS5 under $70",
  "platform": "PS5",
  "genre": "action",
  "maxBudgetCents": 7000,
  "customerId": "cust-001"
}
```

`platform`, `genre`, and `maxBudgetCents` are optional. `query` and `customerId` are required.

### FollowUpRequest (request body)

```json
{
  "refinementText": "Actually, I'd prefer something with a strong single-player story mode"
}
```

### Session (response body)

```json
{
  "sessionId": "s-3ac...",
  "preferences": {
    "query": "I want a co-op action game for two players on PS5 under $70",
    "platform": "PS5",
    "genre": "action",
    "maxBudgetCents": 7000,
    "sessionId": "s-3ac...",
    "customerId": "cust-001",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "recommendation": {
    "suggestions": [
      {
        "catalogId": "elden-ring-ps5",
        "title": "Elden Ring",
        "platform": "PS5",
        "priceCents": 5999,
        "confidenceScore": 0.95,
        "rationale": "Open-world action RPG with co-op support; within budget.",
        "upsellNote": "Shadow of the Erdtree DLC adds 20+ hours."
      },
      {
        "catalogId": "god-of-war-ragnarok-ps5",
        "title": "God of War Ragnarök",
        "platform": "PS5",
        "priceCents": 6999,
        "confidenceScore": 0.88,
        "rationale": "Award-winning action-adventure at the budget ceiling.",
        "upsellNote": null
      }
    ],
    "summary": "Two strong action titles for PS5 within your $70 budget.",
    "decidedAt": "2026-06-28T14:00:12Z"
  },
  "followUpRecommendation": null,
  "status": "RECOMMENDED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:12Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-3ac...", "status": "RECOMMENDED", "recommendation": { ... }, ... }
```

One event per state transition (`QUERYING`, `RECOMMENDED`, `FOLLOW_UP_QUERYING`, `FOLLOW_UP_RECOMMENDED`, `FAILED`). Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `customerId` from the authenticated principal rather than the request body.
