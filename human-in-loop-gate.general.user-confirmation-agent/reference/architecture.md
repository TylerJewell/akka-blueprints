# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: plan, await confirmation, execute.

## Component graph

`ConfirmationEndpoint` accepts a request description and starts `ConfirmationWorkflow` with a fresh request id. The workflow drives `PlannerAgent` to produce an action plan, writes the plan to `RequestEntity`, then waits at a confirmation task. A human calls confirm or cancel through the endpoint, which transitions `RequestEntity`. On confirmation the workflow drives `ExecutorAgent` and records the execution result. `RequestEntity` events project into `RequestsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs description → plan → pause → human confirm → execute. The await-confirmation task does not hold a thread: `awaitConfirmationStep` reads `RequestEntity.getRequest`, and while the status is `PENDING_CONFIRMATION` it self-schedules a 5-second resume timer. When the human transitions the status to `CONFIRMED`, the next poll moves the workflow to `executeStep`. A `CANCELLED` status ends the workflow without execution.

## State machine

A request moves `PENDING_CONFIRMATION → CONFIRMED → EXECUTED` on the success path, or `PENDING_CONFIRMATION → CANCELLED` when the user declines. `CONFIRMED` and the two terminal states are reached only through their corresponding events (`RequestConfirmed`, `RequestExecuted`, `RequestCancelled`).

## Entity model

`RequestEntity` is event-sourced; each command emits one event that the applier folds into the `Request` state. `RequestsView` projects the same events into a row keyed by request id. There is one view query, `getAllRequests`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
