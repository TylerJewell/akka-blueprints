# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: propose, await approval, execute.

## Component graph

`ApprovalEndpoint` accepts an operation request and starts `ApprovalWorkflow` with a fresh action id. The workflow drives `ProposalAgent` to analyze the request and return a structured proposal, writes the proposal to `ActionEntity`, then waits at an approval task. An operator calls approve or deny through the endpoint, which transitions `ActionEntity`. On approval the workflow drives `ExecutionAgent`, which simulates the operation and records the result. `ActionEntity` events project into `ActionsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs: operation request → proposal → pause → operator approve → execute. The await-approval task does not hold a thread: `awaitApprovalStep` reads `ActionEntity.getAction`, and while the status is `PROPOSED` it self-schedules a 5-second resume timer. When the operator transitions the status to `APPROVED`, the next poll moves the workflow to `executeStep`. A `DENIED` status ends the workflow without executing.

## State machine

An action moves `PROPOSED → APPROVED → EXECUTED` on the success path, or `PROPOSED → DENIED` when the operator declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`ActionApproved`, `ActionExecuted`, `ActionDenied`). The `PROPOSED → APPROVED` transition does not itself execute anything; the execution happens only when `executeStep` runs inside the workflow after the next poll confirms `APPROVED`.

## Entity model

`ActionEntity` is event-sourced; each command emits one event that the applier folds into the `Action` state. `ActionsView` projects the same events into a row keyed by action id. There is one view query, `getAllActions`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2). The view row type is `Action`, which is also the entity state record, so all nullable lifecycle fields use `Optional<T>` (Lesson 6).
