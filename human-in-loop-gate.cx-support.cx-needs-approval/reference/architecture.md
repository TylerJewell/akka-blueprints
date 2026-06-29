# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: triage, await approval, resolve.

## Component graph

`SupportEndpoint` accepts a customer message and starts `SupportWorkflow` with a fresh ticket id. The workflow drives `TriageAgent` to analyze the ticket and draft a response, writes the draft to `TicketEntity`, then waits at an approval task. A supervisor calls approve or reject through the endpoint, which transitions `TicketEntity`. On approval the workflow drives `ResolutionAgent` and records the resolution confirmation. `TicketEntity` events project into `TicketsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs customer-message → triage → pause → supervisor-approve → resolve. The await-approval task does not hold a thread: `awaitApprovalStep` reads `TicketEntity.getTicket`, and while the status is `TRIAGED` it self-schedules a 5-second resume timer. When the supervisor transitions the status to `APPROVED`, the next poll moves the workflow to `resolveStep`. A `REJECTED` status ends the workflow with no delivery.

## State machine

A ticket moves `TRIAGED → APPROVED → RESOLVED` on the success path, or `TRIAGED → REJECTED` when the supervisor declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`TicketApproved`, `TicketResolved`, `TicketRejected`).

## Entity model

`TicketEntity` is event-sourced; each command emits one event that the applier folds into the `Ticket` state. `TicketsView` projects the same events into a row keyed by ticket id. There is one view query, `getAllTickets`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
