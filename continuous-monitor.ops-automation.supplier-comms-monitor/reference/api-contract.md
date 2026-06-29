# API contract — supplier-comms-monitor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/orders` | — | `200 [ PurchaseOrder... ]` (sorted newest-first) | `PoOrderEndpoint` ← `PoOrderView` |
| `GET` | `/api/orders/{id}` | — | `200 PurchaseOrder` / `404` | `PoOrderEndpoint` ← `PoOrderView` |
| `POST` | `/api/orders/{id}/approve` | `{ "decidedBy": String }` | `200 PurchaseOrder` (status now OUTREACH_SENT) | `PoOrderEndpoint` → `PurchaseOrderEntity` |
| `POST` | `/api/orders/{id}/escalate` | `{ "decidedBy": String, "escalationNote": String }` | `200 PurchaseOrder` (status now PROCUREMENT_REVIEW) | `PoOrderEndpoint` → `PurchaseOrderEntity` |
| `GET` | `/api/orders/sse` | — | `text/event-stream` | `PoOrderEndpoint` ← `PoOrderView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### PurchaseOrder

```json
{
  "poId": "PO-1042",
  "feedEvent": {
    "poId": "PO-1042",
    "supplierId": "SUP-09",
    "supplierName": "Meridian Components Ltd",
    "itemDescription": "Circuit board assembly v3",
    "orderedQty": 500,
    "promisedDate": "2026-07-15",
    "currency": "USD",
    "unitPrice": 24.50,
    "feedTimestamp": "2026-06-28T08:00:00Z"
  },
  "riskAssessment": {
    "tier": "AT_RISK",
    "confidence": "high",
    "rationale": "Short lead time on high-volume single-source order.",
    "riskFactors": ["short-lead-time", "single-source"]
  },
  "outreachDraft": {
    "subjectLine": "Delivery Confirmation: Circuit board assembly v3",
    "body": "...",
    "requiresBuyerApproval": true,
    "draftedAt": "2026-06-28T08:00:12Z"
  },
  "buyerDecision": {
    "approved": true,
    "decidedBy": "buyer-7",
    "escalationNote": null,
    "decidedAt": "2026-06-28T08:03:40Z"
  },
  "accuracyScore": null,
  "status": "OUTREACH_SENT",
  "createdAt": "2026-06-28T08:00:00Z",
  "closedAt": null
}
```

### SSE event format

```
event: order-update
data: { "poId": "PO-1042", "status": "AWAITING_BUYER_APPROVAL", ... }
```

One event per state transition. Clients reconcile by `poId`.

## Notes

- `GET /api/orders` returns all POs regardless of status; callers filter client-side.
- Approve is only valid when `status == AWAITING_BUYER_APPROVAL`; the endpoint returns `409` otherwise.
- Escalate is valid from `AWAITING_BUYER_APPROVAL`; `escalationNote` is required.
