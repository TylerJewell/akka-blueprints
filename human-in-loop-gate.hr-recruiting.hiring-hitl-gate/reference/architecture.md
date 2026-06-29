# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: propose, await validator, record outcome.

## Component graph

`HiringEndpoint` accepts a candidate application and starts `HiringWorkflow` with a fresh application id. The workflow drives `HiringDecisionProposer` to evaluate the candidate and `MeetingProposer` to draft an interview invitation. Both results are written to `ApplicationEntity` as a single `recordProposals` command; the entity transitions to `PROPOSED` and the workflow pauses at an await-validator task. A hiring manager calls approve or decline through the endpoint, which transitions `ApplicationEntity`. On approval, the workflow invokes `DecisionsReachedService` (modelled as the outcome-recording step) and writes `recordOutcome`, moving the application to `DECIDED`. `ApplicationEntity` events project into `ApplicationsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs submit → propose → pause → human approve → record outcome. The await-validator task does not hold a thread: `awaitValidatorStep` reads `ApplicationEntity.getApplication`, and while the status is `PROPOSED` it self-schedules a 5-second resume timer. When the hiring manager transitions the status to `APPROVED`, the next poll moves the workflow to `recordOutcomeStep`. A `DECLINED` status ends the workflow without recording an outcome.

## State machine

An application moves `PROPOSED → APPROVED → DECIDED` on the success path, or `PROPOSED → DECLINED` when the validator declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`ApplicationApproved`, `OutcomeRecorded`, `ApplicationDeclined`).

## Entity model

`ApplicationEntity` is event-sourced; each command emits one event that the applier folds into the `Application` state. `ApplicationsView` projects the same events into a row keyed by application id. There is one view query, `getAllApplications`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
