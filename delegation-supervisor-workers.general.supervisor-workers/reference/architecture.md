# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A task enters through `CollaborationEndpoint` (`POST /api/tasks`) or is dripped by `TaskSimulator` every 60 seconds. Either path writes a `TaskSubmitted` event onto `TaskRequestQueue`. `TaskRequestConsumer` subscribes to those events and starts one `CollaborationWorkflow` per submission, keyed by `taskId`.

The workflow is the supervisor. It asks `TaskSupervisor` to classify and route the incoming task, fans the work out to `ResearchWorker` and `ChartWorker` in parallel, then asks `TaskSupervisor` to synthesise. Every transition is written as a command to `TaskEntity`, whose events project into `TaskView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a task to score.

The `AppEndpoint` serves the embedded single-file UI from `static-resources/` and the `/api/metadata/*` endpoints that power the Risk Survey and Eval Matrix tabs.

## Interaction sequence

The sequence diagram traces the primary journey: route, parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results before calling the synthesis step. The guardrail decides between the `COMPLETED` and `BLOCKED` terminals.

The supervisor is called twice: once for routing at the start and once for synthesis at the end. The routing call is lightweight (classification only); the synthesis call carries the full worker payloads and gets a 90-second timeout.

## State machine

`CollaborationTask` moves `ROUTING → IN_PROGRESS`, then to one of three terminals: `COMPLETED` (guardrail passed), `DEGRADED` (a worker timed out), or `BLOCKED` (guardrail failed). `COMPLETED` accepts one further `TaskEvalScored` self-transition when `EvalSampler` records a score.

The DEGRADED path does not retry. If `researchStep` or `chartStep` exceeds 60 seconds, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output returned and calls `TaskEntity.degradeTask`.

## Entity model

`TaskRequestQueue` seeds one `CollaborationTask` per task description. A task owns at most one `RoutingDecision`, one `ResearchOutput`, one `ChartData`, and one `TaskResult`. The view row (`CollaborationTaskRow`) mirrors the task with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
