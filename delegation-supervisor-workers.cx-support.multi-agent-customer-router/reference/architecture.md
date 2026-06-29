# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A ticket enters through `TicketEndpoint` (`POST /api/tickets`) or is dripped by `TicketSimulator` every 60 seconds. Either path writes a `TicketSubmitted` event onto `TicketQueueEntity`. `TicketQueueConsumer` subscribes to those events and starts one `TicketWorkflow` per submission, keyed by `ticketId`.

The workflow is the supervisor. It calls `RouterAgent` to classify the ticket, then runs a guardrail that audits the routing decision before any specialist is called. If the guardrail passes, the workflow branches by category to exactly one of `BillingAgent`, `TechnicalAgent`, or `AccountAgent`. Every lifecycle transition is written as a command to `TicketEntity`, whose events project into `TicketView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a resolved ticket to score.

## Interaction sequence

The sequence diagram traces the primary journey: open, route, guardrail check, delegate to specialist, resolve. The guardrail gate separates routing from delegation — no specialist runs if the classification is rejected. The `alt` block shows both the happy path (RESOLVED) and the guardrail failure path (REVIEW_REQUIRED).

## State machine

`Ticket` moves `OPEN → ROUTING`, then either to `REVIEW_REQUIRED` (guardrail failed) or `IN_PROGRESS` (guardrail passed). From `IN_PROGRESS` the ticket moves to `RESOLVED` (specialist answered) or `ESCALATED` (specialist timed out). `RESOLVED` accepts one further `ResolutionEvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`TicketQueueEntity` seeds one `Ticket` per submission. A ticket owns at most one `RoutingDecision` (captured in `category` and `routingRationale` fields) and one `Resolution`. The view row mirrors the ticket with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
