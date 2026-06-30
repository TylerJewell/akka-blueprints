# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: score, await review, qualify.

## Component graph

`LeadScoringEndpoint` accepts a lead profile and starts `LeadScoringWorkflow` with a fresh lead id. The workflow drives `ScoringAgent` to score the lead, writes the result to `LeadEntity`, emits an eval event recording the decision, then waits at a review task. A human calls approve or reject through the endpoint, which transitions `LeadEntity`. On approval the workflow drives `QualificationAgent` and records the qualification summary. `LeadEntity` events project into `LeadsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs profile → score → pause → human approve → qualify. The await-review task does not hold a thread: `awaitReviewStep` reads `LeadEntity.getLead`, and while the status is `SCORED` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED`, the next poll moves the workflow to `qualifyStep`. A `DISQUALIFIED` status ends the workflow.

## State machine

A lead moves `SCORED → APPROVED → QUALIFIED` on the success path, or `SCORED → DISQUALIFIED` when the reviewer declines. `APPROVED` and the two terminal states are reached only through their corresponding events (`LeadApproved`, `LeadQualified`, `LeadRejected`). The workflow never bypasses the gate: `qualifyStep` begins only after `awaitReviewStep` observes `APPROVED`.

## Entity model

`LeadEntity` is event-sourced; each command emits one event that the applier folds into the `Lead` state. `LeadsView` projects the same events into a row keyed by lead id. There is one view query, `getAllLeads`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2). The eval event emitted after scoring is separate from the entity event stream; it carries the input profile alongside the score for downstream drift analysis.
