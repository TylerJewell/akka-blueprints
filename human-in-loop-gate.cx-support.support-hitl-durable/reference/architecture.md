# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: triage, await operator decision, resolve.

## Component graph

`SupportEndpoint` accepts an incoming ticket subject and body and starts `SupportWorkflow` with a fresh ticket id. The workflow drives `TriageAgent` to classify and summarize the ticket, then writes the sanitized triage result to `TicketEntity`. The workflow then waits at an operator-decision task. An operator calls approve or escalate through the endpoint, which transitions `TicketEntity`. On approval, the workflow drives `ResolutionAgent` to draft the customer response and records the resolution. On escalation, the workflow ends and the ticket is marked terminal. `TicketEntity` events project into `TicketsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs ticket submission → triage → pause → operator approve → resolution. The await-decision task does not hold a thread: `awaitDecisionStep` reads `TicketEntity.getTicket`, and while the status is `TRIAGED` it self-schedules a 5-second resume timer. When the operator transitions the status to `APPROVED`, the next poll moves the workflow to `resolveStep`. An `ESCALATED` status ends the workflow immediately.

## State machine

A ticket moves `TRIAGED → APPROVED → RESOLVED` on the standard path, or `TRIAGED → ESCALATED` when the operator routes it for further handling. `APPROVED` and both terminal states are reached only through their corresponding events (`TicketApproved`, `TicketResolved`, `TicketEscalated`). The workflow never advances to the resolution step unless it observes the `APPROVED` status during its poll cycle.

## Entity model

`TicketEntity` is event-sourced; each command emits one event that the applier folds into the `Ticket` state. `TicketsView` projects the same events into a row keyed by ticket id. There is one view query, `getAllTickets`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2). The PII sanitizer operates at the agent output boundary, so the `triageSummary` field stored on the entity contains no raw customer identifiers.
