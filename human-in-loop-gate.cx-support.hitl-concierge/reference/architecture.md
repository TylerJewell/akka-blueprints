# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: triage, await specialist approval, fulfil.

## Component graph

`ConciergeEndpoint` accepts a customer request and starts `ConciergeWorkflow` with a fresh case id. The workflow drives `TriageAgent` to analyse the request and propose a resolution, writes the triage result to `CaseEntity`, then waits at an approval task. A specialist calls approve or decline through the endpoint, which transitions `CaseEntity`. On approval the workflow drives `FulfillmentAgent` and records the fulfilment confirmation. `CaseEntity` events project into `CasesView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs customer request → triage → pause → specialist approve → fulfil. The await-approval task does not hold a thread: `awaitApprovalStep` reads `CaseEntity.getCase`, and while the status is `TRIAGED` it self-schedules a 5-second resume timer. When the specialist transitions the status to `APPROVED`, the next poll moves the workflow to `fulfilStep`. A `DECLINED` status ends the workflow without fulfilment.

## State machine

A case moves `TRIAGED → APPROVED → FULFILLED` on the success path, or `TRIAGED → DECLINED` when the specialist rejects the proposed resolution. `APPROVED` and the two terminal states are reached only through their corresponding events (`CaseApproved`, `CaseFulfilled`, `CaseDeclined`).

## Entity model

`CaseEntity` is event-sourced; each command emits one event that the applier folds into the `Case` state. `CasesView` projects the same events into a row keyed by case id. There is one view query, `getAllCases`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
