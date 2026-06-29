# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: propose, await approval, execute.

## Component graph

`ActionEndpoint` accepts a request text and starts `ActionWorkflow` with a fresh action id. The workflow drives `ProposalAgent` to analyse the request and produce a structured proposal, writes the proposal to `ActionEntity`, then waits at an approval task. A human calls approve or reject through the endpoint, which transitions `ActionEntity`. On approval the workflow drives `ExecutionAgent` and records the execution outcome. `ActionEntity` events project into `ActionsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs request → propose → pause → human approve → execute. The await-approval task does not hold a thread: `awaitApprovalStep` reads `ActionEntity.getAction`, and while the status is `PROPOSED` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED`, the next poll moves the workflow to `executeStep`. A `REJECTED` status ends the workflow.

## State machine

An action moves `PROPOSED → APPROVED → EXECUTED` on the success path, or `PROPOSED → REJECTED` when the reviewer declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`ActionApproved`, `ActionExecuted`, `ActionRejected`). No event can be emitted out of order; each event-applier checks the current status before accepting a transition.

## Entity model

`ActionEntity` is event-sourced; each command emits one event that the applier folds into the `Action` state. `ActionsView` projects the same events into a row keyed by action id. There is one view query, `getAllActions`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2). The view row type is `Action` itself, so all lifecycle fields must be `Optional<T>` (Lesson 6).
