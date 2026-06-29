# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: validate, await approval, fulfill.

## Component graph

`OrderEndpoint` accepts an order payload and starts `OrderWorkflow` with a fresh order id. The workflow drives `ValidationAgent` to produce a review, writes the review to `OrderEntity`, then waits at an approval task. A human calls approve or reject through the endpoint, which transitions `OrderEntity`. On approval the workflow drives `FulfillmentAgent` and records the fulfillment confirmation. `OrderEntity` events project into `OrdersView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs order submission → validation → pause → human approve → dispatch. The await-approval task does not hold a thread: `awaitApprovalStep` reads `OrderEntity.getOrder`, and while the status is `PENDING_APPROVAL` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED`, the next poll moves the workflow to `fulfillStep`. A `REJECTED` status ends the workflow.

## State machine

An order moves `PENDING_APPROVAL → APPROVED → FULFILLED` on the success path, or `PENDING_APPROVAL → REJECTED` when the reviewer declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`OrderApproved`, `OrderFulfilled`, `OrderRejected`). No direct transition from `PENDING_APPROVAL` to `FULFILLED` is possible; the approval event is the mandatory gate.

## Entity model

`OrderEntity` is event-sourced; each command emits one event that the applier folds into the `Order` state. `OrdersView` projects the same events into a row keyed by order id. There is one view query, `getAllOrders`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2). The entity holds validation output (`lineItemsSummary`, `estimatedTotal`, `riskFlags`) alongside fulfillment output (`trackingId`, `fulfilledAt`), so the UI can display both phases from a single row.
