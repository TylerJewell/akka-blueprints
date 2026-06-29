# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: screen, await compliance-officer approval, close.

## Component graph

`ScreeningEndpoint` accepts an entity id and source document references and starts `ScreeningWorkflow` with a fresh case id. Before the entity file reaches `ScreenerAgent`, a PII sanitizer tokenizes names, ID numbers, and addresses; `ScreenerAgent` operates only on tokenized text. The workflow writes the screening result to `CaseEntity`, then waits at an approval task. A compliance officer calls approve or reject through the endpoint, which transitions `CaseEntity`. On approval the workflow drives `VerdictAgent` and records the case closure. `CaseEntity` events project into `CasesView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs: submit entity → screen → pause → compliance-officer approve → close. The await-approval task does not hold a thread: `awaitApprovalStep` reads `CaseEntity.getCase`, and while the status is `SCREENED` it self-schedules a 5-second resume timer. When the officer transitions the status to `OFFICER_APPROVED`, the next poll moves the workflow to `closeStep`. A `REJECTED` status ends the workflow.

## State machine

A case moves `SCREENED → OFFICER_APPROVED → CLOSED` on the approval path, or `SCREENED → REJECTED` when the officer declines. `OFFICER_APPROVED` and the two terminal states are reached only through their corresponding events (`CaseOfficerApproved`, `CaseClosed`, `CaseRejected`).

## Entity model

`CaseEntity` is event-sourced; each command emits one event that the applier folds into the `Case` state. `CasesView` projects the same events into a row keyed by case id. There is one view query, `getAllCases`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
