# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 3-task graph in the human-in-loop-gate pattern: plan, await approval, execute.

## Component graph

`PlanEndpoint` accepts a goal and starts `PlanWorkflow` with a fresh plan id. The workflow drives `PlanningAgent` to generate an execution plan, writes the plan to `PlanEntity`, then waits at an approval task. While waiting, the human can read the plan through the UI and optionally call the edit endpoint to revise the steps; `PlanEntity` records the edit as a `PlanEdited` event but keeps status in `PLANNED`. A human then calls approve or cancel through the endpoint, which transitions `PlanEntity`. On approval the workflow drives `ExecutionAgent` and records the execution outcome. `PlanEntity` events project into `PlansView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs goal → plan → pause → optional edit → human approve → execute. The await-approval task does not hold a thread: `awaitApprovalStep` reads `PlanEntity.getPlan`, and while status is `PLANNED` it self-schedules a 5-second resume timer. When the human transitions status to `APPROVED`, the next poll moves the workflow to `executeStep`. A `CANCELLED` status ends the workflow without reaching execution.

The edit path fits into the pause phase: the reviewer PATCHes `/api/plans/{id}/edit`, which calls `PlanEntity.editPlan`; this emits `PlanEdited` and updates the stored steps. The workflow's next poll still sees status `PLANNED` and continues waiting. When the reviewer is satisfied, they approve — the workflow then passes the updated steps to `ExecutionAgent`.

## State machine

A plan moves `PLANNED → APPROVED → EXECUTED` on the success path. It may self-loop on `PLANNED` when edited (a `PlanEdited` event does not change status). It moves to terminal `CANCELLED` when the reviewer declines. `APPROVED` and the two terminal states are only reachable through their corresponding events (`PlanApproved`, `PlanExecuted`, `PlanCancelled`).

## Entity model

`PlanEntity` is event-sourced; each command emits one event that the applier folds into the `Plan` state. `PlansView` projects the same events into a row keyed by plan id. There is one view query, `getAllPlans`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2). The `steps` field on the view row is `Optional<List<String>>` because it is absent until `PlanCreated` is applied (Lesson 6).
