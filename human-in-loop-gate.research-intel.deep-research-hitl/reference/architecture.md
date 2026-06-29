# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 4-step pipeline in the human-in-loop-gate pattern: plan, investigate, synthesise, await review, deliver.

## Component graph

`ResearchEndpoint` accepts a query and starts `ResearchWorkflow` with a fresh report id. The workflow drives `PlannerAgent` to decompose the query into sub-topics and writes the plan to `ResearchEntity`. It then calls `ResearchAgent` once per sub-topic in sequence, accumulating findings. Once all findings are collected, `SynthesisAgent` assembles them into a report draft and the entity transitions to `AWAITING_REVIEW`. A human calls approve or reject through the endpoint, which transitions `ResearchEntity`. On approval the workflow records delivery; on rejection it loops back to `SynthesisAgent` with the reviewer's notes. `ResearchEntity` events project into `ReportsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs query → plan → investigate (fan-out) → synthesise → pause → human approve → deliver. The await-review task does not hold a thread: `awaitReviewStep` reads `ResearchEntity.getReport`, and while the status is `AWAITING_REVIEW` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED`, the next poll moves the workflow to `deliverStep`. A `NEEDS_REVISION` status directs the workflow back to `synthesiseStep` with the reviewer's notes appended to the synthesis context, producing a revised draft that returns to `AWAITING_REVIEW` for another cycle.

## State machine

A report moves `PLANNING → INVESTIGATING → SYNTHESISING → AWAITING_REVIEW → APPROVED → DELIVERED` on the success path. A rejection at `AWAITING_REVIEW` moves the report to `NEEDS_REVISION`, from which the workflow re-enters `SYNTHESISING` and returns to `AWAITING_REVIEW`. Each state is reached only through its corresponding event, and the APPROVED and two terminal states cannot be revisited once entered.

## Entity model

`ResearchEntity` is event-sourced; each command emits one event that the applier folds into the `Report` state. `ReportsView` projects the same events into a row keyed by report id. There is one view query, `getAllReports`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
