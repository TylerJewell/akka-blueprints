# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: process, await approval, finalize.

## Component graph

`TaskEndpoint` accepts a task request and starts `ApprovalWorkflow` with a fresh task id. The workflow drives `TaskProcessorAgent` to analyze the request and produce a recommendation, writes the result to `TaskEntity`, then waits at an approval task. A human calls approve or reject through the endpoint, which transitions `TaskEntity`. On approval the workflow drives `TaskFinalizerAgent` and records the finalization outcome. `TaskEntity` events project into `TasksView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs request → process → pause → human approve → finalize. The await-approval task does not hold a thread: `awaitApprovalStep` reads `TaskEntity.getTask`, and while the status is `PROCESSED` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED`, the next poll moves the workflow to `finalizeStep`. A `REJECTED` status ends the workflow immediately.

## State machine

A task moves `PROCESSED → APPROVED → FINALIZED` on the success path, or `PROCESSED → REJECTED` when the reviewer declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`TaskApproved`, `TaskFinalized`, `TaskRejected`). No direct transition from `PROCESSED` to `FINALIZED` exists — the approval step cannot be skipped.

## Entity model

`TaskEntity` is event-sourced; each command emits one event that the applier folds into the `Task` state. `TasksView` projects the same events into a row keyed by task id. There is one view query, `getAllTasks`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
