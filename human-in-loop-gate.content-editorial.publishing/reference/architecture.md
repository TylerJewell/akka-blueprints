# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: draft, await approval, publish.

## Component graph

`PublishingEndpoint` accepts a topic and starts `PublishingWorkflow` with a fresh post id. The workflow drives `ContentAgent` to draft, writes the draft to `PostEntity`, then waits at an approval task. A human calls approve or reject through the endpoint, which transitions `PostEntity`. On approval the workflow drives `PublishingAgent` and records the publication. `PostEntity` events project into `PostsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs topic → draft → pause → human approve → publish. The await-approval task does not hold a thread: `awaitApprovalStep` reads `PostEntity.getPost`, and while the status is `DRAFTED` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED`, the next poll moves the workflow to `publishStep`. A `REJECTED` status ends the workflow.

## State machine

A post moves `DRAFTED → APPROVED → PUBLISHED` on the success path, or `DRAFTED → REJECTED` when the reviewer declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`PostApproved`, `PostPublished`, `PostRejected`).

## Entity model

`PostEntity` is event-sourced; each command emits one event that the applier folds into the `Post` state. `PostsView` projects the same events into a row keyed by post id. There is one view query, `getAllPosts`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
