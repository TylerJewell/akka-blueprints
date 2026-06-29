# API contract — personalized-shopper

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `StartSessionRequest` | `201 { sessionId }` | `ShoppingEndpoint` → `ShoppingSessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ ShoppingSession... ]` (newest-first) | `ShoppingEndpoint` ← `RecommendationView` |
| `GET` | `/api/sessions/{id}` | — | `200 ShoppingSession` / `404` | `ShoppingEndpoint` ← `RecommendationView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `ShoppingEndpoint` ← `RecommendationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body)

```json
{
  "preferredCategories": ["electronics", "home-goods"],
  "preferredBrands": ["TechCore", "NovaBright"],
  "minPrice": 50.0,
  "maxPrice": 300.0,
  "excludedCategories": ["apparel"],
  "freeTextNote": "I prefer energy-efficient products.",
  "shopperEmail": "alex.morgan@example.com",
  "shopperPhone": "+1-555-0192",
  "loyaltyCardNumber": "LC-77340012",
  "shopperName": "Alex Morgan",
  "catalogSnapshot": "electronics"
}
```

### ShoppingSession (response body)

```json
{
  "sessionId": "s-4c9...",
  "profile": {
    "sessionId": "s-4c9...",
    "preferredCategories": ["electronics", "home-goods"],
    "preferredBrands": ["TechCore", "NovaBright"],
    "minPrice": 50.0,
    "maxPrice": 300.0,
    "excludedCategories": ["apparel"],
    "freeTextNote": "I prefer energy-efficient products.",
    "shopperEmail": "alex.morgan@example.com",
    "shopperPhone": "+1-555-0192",
    "loyaltyCardNumber": "LC-77340012",
    "shopperName": "Alex Morgan",
    "submittedAt": "2026-06-28T14:20:00Z"
  },
  "sanitized": {
    "preferredCategories": ["electronics", "home-goods"],
    "preferredBrands": ["TechCore", "NovaBright"],
    "minPrice": 50.0,
    "maxPrice": 300.0,
    "excludedCategories": ["apparel"],
    "freeTextNote": "I prefer energy-efficient products.",
    "piiCategoriesStripped": ["email", "phone", "loyalty-card", "person-name"]
  },
  "recommendations": {
    "outcome": "MATCHED",
    "explanation": "Three TechCore products fall within budget and are in stock.",
    "recommendations": [
      {
        "rank": 1,
        "productId": "EL-0021",
        "productName": "TechCore Smart Hub",
        "category": "electronics",
        "price": 129.99,
        "rationale": "Preferred brand TechCore, within budget, energy-efficiency tag, currently in season.",
        "confidence": "HIGH"
      }
    ],
    "decidedAt": "2026-06-28T14:20:18Z"
  },
  "freshness": {
    "score": 4,
    "rationale": "3 of 3 recommended products are in stock; 2 of 3 are current season.",
    "scoredAt": "2026-06-28T14:20:19Z"
  },
  "status": "FRESHNESS_SCORED",
  "createdAt": "2026-06-28T14:20:00Z",
  "finishedAt": "2026-06-28T14:20:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

Note: `profile.shopperEmail`, `profile.shopperPhone`, `profile.loyaltyCardNumber`, and `profile.shopperName` appear in the full entity GET response for audit purposes. The `RecommendationView` row (`SessionRow`) omits these fields; the UI only displays the sanitized profile. A consumer needing the raw PII fields fetches `GET /api/sessions/{id}` and reads from `profile.*` directly.

### SSE event format

```
event: session-update
data: { "sessionId": "s-4c9...", "status": "RECOMMENDATIONS_READY", "recommendations": { ... }, ... }
```

One event per state transition (`STARTED`, `PROFILE_SANITIZED`, `RECOMMENDING`, `RECOMMENDATIONS_READY`, `FRESHNESS_SCORED`, `FAILED`). Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and populate `shopperEmail` / `shopperName` from the authenticated principal rather than the request body — or strip those fields entirely if the identity system provides a non-PII shopper token.
