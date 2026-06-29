# API contract — sales-offer-generator

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## OfferEndpoint (`/api`)

### `POST /api/offers`
Submit a customer brief.
- Body: `{ "brief": "string" }`
- Response: `{ "offerId": "uuid" }`
- Produces: enqueues `BriefQueued` on `InboundRequestQueue`.

### `GET /api/offers?status=...`
List offers, optionally filtered by status (filter applied client-side over `getAllOffers`).
- Response: `{ "offers": [Offer, ...] }`
- Consumes: `OffersView.getAllOffers`.

### `GET /api/offers/{offerId}`
Single offer.
- Response: `Offer` or 404.

### `GET /api/offers/sse`
Server-Sent Events stream of `Offer` updates.
- Content-Type: `text/event-stream`
- Each event `data:` is one JSON `Offer` (see below).

## CatalogEndpoint (`/api/catalog`)

### `GET /api/catalog?segment=...`
In-process product catalog. Returns canned products for the segment.
- Response: `{ "products": [ { "sku": "string", "name": "string", "listPrice": 0.0, "segment": "string" }, ... ] }`
- Consumed by `ProductMatcher`'s catalog tool.

## Metadata (`/api/metadata`)

```
GET /api/metadata/eval-matrix   -> text/yaml
GET /api/metadata/risk-survey   -> text/yaml
GET /api/metadata/readme        -> text/markdown
```
Served from `src/main/resources/metadata/`.

## AppEndpoint

```
GET /              -> 302 /app/index.html
GET /app/{*path}   -> static-resources/{*path}
```

## `Offer` JSON form

Lifecycle fields are nullable (`Optional` in Java; raw value or `null` on the wire).

```json
{
  "id": "uuid",
  "brief": "string or null",
  "status": "QUEUED | ANALYZED | MATCHED | PRICED | COMPOSED | APPROVED | REJECTED",
  "analyzedAt": "ISO-8601 or null",
  "needs": { "segment": "string", "painPoints": ["string"], "budgetBand": "string", "desiredOutcomes": ["string"] },
  "matchedAt": "ISO-8601 or null",
  "products": { "products": [ { "sku": "string", "name": "string", "listPrice": 0.0, "fitReason": "string" } ] },
  "pricedAt": "ISO-8601 or null",
  "pricing": { "subtotal": 0.0, "discountPct": 0.0, "total": 0.0, "rationale": "string" },
  "composedAt": "ISO-8601 or null",
  "offerTitle": "string or null",
  "offerBody": "string or null",
  "quotedTotal": 0.0,
  "approvedAt": "ISO-8601 or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectReason": "string or null"
}
```

Nested `needs`, `products`, and `pricing` are `null` until their stage completes.
