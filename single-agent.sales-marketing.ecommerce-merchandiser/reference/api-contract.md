# API contract — merchandiser

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/proposals` | `SubmitObjectiveRequest` | `201 { proposalId }` | `ProposalEndpoint` → `ProposalEntity` |
| `GET` | `/api/proposals` | — | `200 [ Proposal... ]` (newest-first) | `ProposalEndpoint` ← `ProposalView` |
| `GET` | `/api/proposals/{id}` | — | `200 Proposal` / `404` | `ProposalEndpoint` ← `ProposalView` |
| `POST` | `/api/proposals/{id}/approve` | `ApprovalRequest` | `200 { proposalId, status }` | `ProposalEndpoint` → `ProposalEntity` |
| `POST` | `/api/proposals/{id}/reject` | `ApprovalRequest` | `200 { proposalId, status }` | `ProposalEndpoint` → `ProposalEntity` |
| `GET` | `/api/proposals/sse` | — | `text/event-stream` | `ProposalEndpoint` ← `ProposalView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitObjectiveRequest (request body)

```json
{
  "objectiveText": "Refresh product descriptions in the Home category for the summer season",
  "scope": {
    "kind": "CATEGORY",
    "categorySlug": "home"
  },
  "submittedBy": "merchandiser-alice"
}
```

A `SKU_LIST` scope example:

```json
{
  "objectiveText": "Create a clearance promotion for slow-moving skincare items",
  "scope": {
    "kind": "SKU_LIST",
    "skuList": ["skin-003", "skin-007", "skin-011"]
  },
  "submittedBy": "merchandiser-bob"
}
```

### ApprovalRequest (request body for approve and reject)

```json
{
  "decidedBy": "merchandiser-alice",
  "note": "Approved — descriptions look accurate; launching for summer campaign."
}
```

`note` is optional.

### Proposal (response body)

```json
{
  "proposalId": "p-7ac...",
  "objective": {
    "proposalId": "p-7ac...",
    "objectiveText": "Refresh product descriptions in the Home category for the summer season",
    "scope": { "kind": "CATEGORY", "categorySlug": "home" },
    "submittedBy": "merchandiser-alice",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "context": {
    "products": [
      {
        "sku": "home-001",
        "name": "Outdoor Dining Table",
        "currentDescription": "Durable outdoor table. Available in three colors.",
        "currentPromotion": null,
        "categorySlug": "home",
        "price": 249.99
      }
    ],
    "categoryNames": ["Home", "Apparel", "Electronics"],
    "activePromotionCount": 2,
    "fetchedAt": "2026-06-28T14:00:01Z"
  },
  "proposal": {
    "summary": "Both Home products have generic descriptions that omit seasonal context. Recommend updating both to emphasize summer use cases.",
    "changes": [
      {
        "type": "DESCRIPTION_UPDATE",
        "targetRef": "home-001",
        "currentValue": "Durable outdoor table. Available in three colors.",
        "proposedValue": "Weather-resistant outdoor dining table built for summer entertaining. UV-stable finish in three colors.",
        "rationale": "Current description omits outdoor durability and seasonal context that summer shoppers search for.",
        "confidence": "HIGH"
      }
    ],
    "overallConfidence": "HIGH",
    "proposedAt": "2026-06-28T14:00:18Z"
  },
  "approval": {
    "status": "APPROVED",
    "decidedBy": "merchandiser-alice",
    "note": "Approved — descriptions look accurate; launching for summer campaign.",
    "decidedAt": "2026-06-28T14:05:00Z"
  },
  "status": "PUBLISHED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:05:30Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: proposal-update
data: { "proposalId": "p-7ac...", "status": "PENDING_APPROVAL", "proposal": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `CONTEXT_LOADED`, `GENERATING`, `PENDING_APPROVAL`, `PUBLISHING`, `PUBLISHED`, `REJECTED`, `EXPIRED`, `FAILED`). Clients reconcile by `proposalId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` / `decidedBy` from the authenticated principal rather than the request body. The approval endpoints in particular should be restricted to users with a `merchandiser` or `catalog-admin` role.
