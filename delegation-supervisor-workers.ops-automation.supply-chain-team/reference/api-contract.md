# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/supply` | `{ "sku": "string", "targetQuantity": number }` | `{ "orderId": "uuid" }` | `SupplyEndpoint` → `OrderQueueEntity` |
| GET | `/api/supply` | — | `{ "orders": [SupplyOrderRow, ...] }` | `SupplyEndpoint` → `SupplyOrderView` |
| GET | `/api/supply?status=RECOMMENDED` | — | filtered list (client-side filter) | `SupplyEndpoint` |
| GET | `/api/supply/{id}` | — | `SupplyOrderRow` or 404 | `SupplyEndpoint` |
| GET | `/api/supply/sse` | — | `text/event-stream` of `SupplyOrderRow` | `SupplyEndpoint` → `SupplyOrderView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `SupplyEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `SupplyEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `SupplyEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/supply` request:

```json
{ "sku": "SKU-ELEC-9821", "targetQuantity": 500 }
```

`SupplyOrderRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "orderId": "uuid",
  "sku": "string",
  "targetQuantity": 500,
  "status": "PENDING | ANALYZING | RECOMMENDED | DEGRADED | BLOCKED",
  "stockAssessment": {
    "positions": [
      { "sku": "SKU-ELEC-9821", "onHand": 120, "inTransit": 80, "reorderPoint": 100,
        "stockOutRisk": true, "assessedAt": "ISO-8601" }
    ],
    "overallRisk": "medium",
    "assessedAt": "ISO-8601"
  },
  "routePlan": {
    "segments": [
      { "origin": "Chicago-DC", "destination": "Dallas-Hub", "transitDays": 2,
        "carrier": "FedEx Freight", "costUsd": 1240.50 }
    ],
    "totalTransitDays": 2,
    "totalCostUsd": 1240.50,
    "plannedAt": "ISO-8601"
  },
  "recommendation": {
    "summary": "string",
    "guardrailVerdict": "ok",
    "synthesisedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/supply/sse` emits one event per order change:

```
event: order
data: { ...SupplyOrderRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `orderId`. No polling.
