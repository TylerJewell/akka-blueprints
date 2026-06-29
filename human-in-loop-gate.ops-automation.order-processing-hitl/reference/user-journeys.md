# User journeys

Acceptance journeys the generated system must pass.

## J1 — Submit an order and receive a validation review

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/orders` with `{ "customerId": "cust-001", "lineItems": "10x Widget A @ $12.50, 5x Widget B @ $24.00" }`.
- **Expected:** response returns `{ orderId }`. Within ~30 s the order appears via SSE in `PENDING_APPROVAL` with non-empty `lineItemsSummary` and `estimatedTotal`. Approve/Reject buttons show on the App UI card.

## J2 — Approve and fulfill an order

- **Preconditions:** an order in `PENDING_APPROVAL` with a review (J1).
- **Steps:** `POST /api/orders/{orderId}/approve` with `{ "approvedBy": "ops-manager", "note": "Looks correct" }`.
- **Expected:** `OrderEntity` transitions to `APPROVED`; the workflow's next poll moves to `fulfillStep`; `FulfillmentAgent` runs; within ~30 s status becomes `FULFILLED` with a non-null `trackingId` shown in the UI.

## J3 — Reject an order

- **Preconditions:** an order in `PENDING_APPROVAL` (J1).
- **Steps:** `POST /api/orders/{orderId}/reject` with `{ "rejectedBy": "ops-manager", "reason": "Incorrect quantities — resubmit" }`.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown in the UI card; the workflow ends; the fulfillment step never runs; no `trackingId` is set.

## J4 — Fulfillment guard blocks premature dispatch

- **Preconditions:** an order in `PENDING_APPROVAL` (not yet approved).
- **Steps:** drive the workflow toward `fulfillStep` without an approval (e.g., the await-approval poll runs while status is still `PENDING_APPROVAL`).
- **Expected:** the before-tool-call guardrail re-reads `OrderEntity.status`; because it is not `APPROVED`, the simulated dispatch tool does not run and no `trackingId` is set. The order stays in `PENDING_APPROVAL` until a human acts.

## J5 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 2 controls (H1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders rendered muted italic; the Overview README card renders correctly.

## J6 — SSE stream delivers live updates

- **Preconditions:** UI open on the App UI tab; an order submitted (J1).
- **Steps:** while watching the order list, approve the order from a second browser tab or via curl.
- **Expected:** within 5 s the order card in the first tab updates to `APPROVED` without a page refresh; after ~30 s it updates again to `FULFILLED` with the tracking id, all driven by the SSE stream.
