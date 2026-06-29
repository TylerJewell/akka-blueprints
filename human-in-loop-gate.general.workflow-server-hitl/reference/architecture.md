# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: analyze, await approval, complete. The workflow is exposed as a REST API with Server-Sent Events streaming job state to connected clients.

## Component graph

`WorkflowEndpoint` accepts a job request payload and starts `JobWorkflow` with a fresh job id. The workflow drives `AnalysisAgent` to evaluate the request and classify a risk level, writes the analysis to `JobEntity`, then waits at an approval task. A human reviewer calls approve or reject through the endpoint, which transitions `JobEntity`. On approval the workflow drives `CompletionAgent` and records the result. `JobEntity` events project into `JobsView`, which the endpoint queries and streams over SSE to any connected client. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs job submission → analysis → pause → human approve → completion. The await-approval task does not hold a thread: `awaitApprovalStep` reads `JobEntity.getJob`, and while the status is `ANALYZED` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED`, the next poll moves the workflow to `completeStep`. A `REJECTED` status ends the workflow without running the completion step.

SSE events from `JobsView` keep the UI current throughout all transitions without the client needing to poll. The `WorkflowEndpoint` SSE endpoint streams each `JobsView` row update as it arrives.

## State machine

A job moves `ANALYZED → APPROVED → COMPLETED` on the success path, or `ANALYZED → REJECTED` when the reviewer declines. `APPROVED` and the two terminal states (`COMPLETED`, `REJECTED`) are reached only through their corresponding events (`JobApproved`, `JobCompleted`, `JobRejected`). There is no way to move a job backward through the state machine.

## Entity model

`JobEntity` is event-sourced; each command emits one event that the applier folds into the `Job` state. `JobsView` projects the same events into a row keyed by job id. There is one view query, `getAllJobs`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2). The view row type is `Job`, identical to the entity state record, so no separate projection record is needed.
