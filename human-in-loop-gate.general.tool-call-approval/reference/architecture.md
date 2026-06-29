# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: plan a tool call, await human decision, execute if approved.

## Component graph

`ApprovalEndpoint` accepts a goal and starts `ApprovalWorkflow` with a fresh request id. The workflow drives `PlannerAgent` to select a tool and build parameters, writes the plan to `ToolRequestEntity`, then waits at an approval task. A human operator calls approve (optionally supplying edited parameters) or reject through the endpoint, which transitions `ToolRequestEntity`. On approval the workflow reads the effective parameters from the entity, drives `ExecutorAgent`, and records the execution result. `ToolRequestEntity` events project into `ToolRequestsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs goal → plan → pause → human approve → execute. The await-approval task does not hold a thread: `awaitApprovalStep` reads `ToolRequestEntity.getRequest`, and while the status is `PENDING_APPROVAL` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED` (with or without edited parameters), the next poll moves the workflow to `executeStep`. A `REJECTED` status ends the workflow without executing anything.

The parameter-editing path is transparent to the executor: `executeStep` always reads the entity to obtain effective parameters rather than carrying state from `planStep`. If the operator supplied `editedParameters`, those replace the planner's original values in the `ToolCallApproved` event applier before the executor sees them.

## State machine

A request moves `PENDING_APPROVAL → APPROVED → EXECUTED` on the success path, or `PENDING_APPROVAL → REJECTED` when the operator declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`ToolCallApproved`, `ToolCallExecuted`, `ToolCallRejected`).

## Entity model

`ToolRequestEntity` is event-sourced; each command emits one event that the applier folds into the `ToolRequest` state. `ToolRequestsView` projects the same events into a row keyed by request id. There is one view query, `getAllRequests`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
