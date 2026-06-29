# API contract — steered-renewal-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/renewals` | `SubmitRenewalRequest` | `201 { renewalId }` | `RenewalEndpoint` → `LoanEntity` + `RenewalWorkflow` |
| `GET` | `/api/renewals` | — | `200 [ Renewal... ]` (newest-first) | `RenewalEndpoint` ← `RenewalView` |
| `GET` | `/api/renewals/{id}` | — | `200 Renewal` / `404` | `RenewalEndpoint` ← `RenewalView` |
| `GET` | `/api/renewals/sse` | — | `text/event-stream` | `RenewalEndpoint` ← `RenewalView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRenewalRequest (request body)

```json
{
  "patronId": "patron-001",
  "loanId": "loan-007"
}
```

The endpoint mints a `renewalId` (UUID) server-side.

### Renewal (response body)

```json
{
  "renewalId": "rn-4a2...",
  "request": {
    "renewalId": "rn-4a2...",
    "patronId": "patron-001",
    "loanId": "loan-007",
    "requestedAt": "2026-06-28T14:00:00Z"
  },
  "patron": {
    "patronId": "patron-001",
    "displayName": "Alex Rivera",
    "patronTier": "STANDARD",
    "outstandingFinesCents": 0,
    "lifetimeRenewals": 12
  },
  "loan": {
    "loanId": "loan-007",
    "itemBarcode": "BC-00391",
    "itemTitle": "Designing Data-Intensive Applications",
    "itemType": "BOOK",
    "originalDueDate": "2026-06-20T00:00:00Z",
    "priorRenewalCount": 0,
    "maxRenewalsAllowed": 3
  },
  "decision": {
    "outcome": "APPROVED",
    "reason": "Renewal approved; no outstanding fines and renewal limit not reached.",
    "newDueDate": "2026-07-12T00:00:00Z",
    "decidedAt": "2026-06-28T14:00:18Z"
  },
  "notification": {
    "channel": "in-app",
    "message": "Your renewal for 'Designing Data-Intensive Applications' has been approved. New due date: 12 Jul 2026.",
    "sentAt": "2026-06-28T14:00:19Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: renewal-update
data: { "renewalId": "rn-4a2...", "status": "DECISION_RECORDED", "decision": { ... }, ... }
```

One event per state transition (`REQUESTED`, `ENRICHED`, `DECIDING`, `DECISION_RECORDED`, `COMPLETED`, `FAILED`). Clients reconcile by `renewalId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and validate that the requesting principal is the patron identified by `patronId` (or a library-staff principal with broader access).
