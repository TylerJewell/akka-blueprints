# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A change request enters through `OpsEndpoint` (`POST /api/ops/changes`) or is dripped by `PipelineSimulator` every 90 seconds. Either path writes a `ChangeSubmitted` event onto `PipelineQueue`. `PipelineConsumer` subscribes to those events and starts one `OpsWorkflow` per submission, keyed by `changeId`.

The workflow is the supervisor. It asks `OpsCoordinator` to parse the request into a `WorkPlan`, fans the work out to `InfraAgent`, `DeployAgent`, and `ObservabilityAgent` in parallel, then asks `OpsCoordinator` to consolidate the three assessments into a `ReadinessReport`. Every lifecycle transition is written as a command to `ChangeRequestEntity`, whose events project into `OpsView`. The endpoint reads and streams the view.

`HaltSignalEntity` holds active halt signals issued by operators via `POST /api/ops/halt`. `HaltMonitor` checks for active signals every 30 seconds and forwards halt commands to any in-flight workflow.

## Interaction sequence

The sequence diagram traces the primary journey: parse, parallel fan-out to three specialists, join, consolidate, guardrail, approval gate (for production targets), and persist. The `par` block is the delegation core — all three specialists run concurrently and the workflow joins their results. The approval gate shows the production path; non-production targets skip it.

## State machine

`ChangeRequest` moves `PLANNING → IN_PROGRESS`, then to one of several terminals or an intermediate hold. The production path visits `AWAITING_APPROVAL` before `APPROVED` or `REJECTED`. The failure paths — `DEGRADED` (timeout), `BLOCKED` (guardrail), and `HALTED` (operator signal) — are reachable from `IN_PROGRESS` and, for halt, from `AWAITING_APPROVAL`.

## Entity model

`PipelineQueue` seeds one `ChangeRequest` per submission. A change request owns at most one `WorkPlan`, one `InfraAssessment`, one `DeployAssessment`, one `ObsAssessment`, and one `ReadinessReport`. `HaltSignalEntity` holds a list of active signals; each signal can affect multiple change requests. The view row mirrors the change request with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
