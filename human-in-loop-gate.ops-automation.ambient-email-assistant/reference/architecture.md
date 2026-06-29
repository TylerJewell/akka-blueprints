# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 4-task graph in the human-in-loop-gate pattern: triage, draft action, await approval, execute action.

## Component graph

`EmailEndpoint` accepts an incoming email payload and starts `EmailWorkflow` with a fresh thread id. The workflow drives `TriageAgent` to classify the email by category and urgency, then routes to `ReplyDraftAgent` or `MeetingSchedulerAgent` based on the suggested action. The draft is persisted on `EmailThreadEntity` and the workflow pauses at an approval gate. An operator calls approve or dismiss through the endpoint, which transitions the entity. On approval the workflow executes the action (simulated Gmail send or Calendar create) and records the result. `EmailThreadEntity` events project into `ThreadsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs: receive → triage → draft → pause → operator approve → execute action. The await-approval task does not hold a thread: `awaitApprovalStep` reads `EmailThreadEntity.getThread` and while status is `ACTION_DRAFTED` it self-schedules a 5-second resume timer. When the operator transitions status to `APPROVED`, the next poll drives the workflow to `executeActionStep`. A `DISMISSED` status ends the workflow immediately with no outbound action.

The draft-action step branches on the `suggestedAction` field from the triage classification: `reply` routes to `ReplyDraftAgent`, `meeting` routes to `MeetingSchedulerAgent`, and any other value defaults to `ReplyDraftAgent`. This branch happens inside the workflow and is invisible to the entity, which only cares about the resulting `ActionDrafted` event.

## State machine

A thread moves `RECEIVED → TRIAGED → ACTION_DRAFTED → APPROVED → REPLY_SENT` (or `MEETING_SCHEDULED`) on the success path. `ACTION_DRAFTED → DISMISSED` is the operator-decline path; it is terminal with no outbound action. `APPROVED` is an intermediate state that lasts only until `executeActionStep` completes; callers should not rely on observing it in the SSE stream.

## Entity model

`EmailThreadEntity` is event-sourced; each command emits one event that the applier folds into the `EmailThread` state. `ThreadsView` projects the same events into a row keyed by thread id. There is one view query, `getAllThreads`; callers filter by status or category client-side because Akka cannot auto-index enum columns (Lesson 2).

The PII sanitizer operates upstream of the entity: it strips PII from the email payload before the payload enters any agent prompt. The originals are held in `EmailThreadEntity` under command-handler access control and are not projected into the view.
