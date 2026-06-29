# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A task enters through `TaskEndpoint` (`POST /api/tasks`) or is dripped by `TaskSimulator` every 60 seconds. Either path writes a `TaskSubmitted` event onto `TaskQueue`. `TaskRequestConsumer` subscribes to those events and starts one `TaskOrchestrationWorkflow` per submission, keyed by `taskId`.

The workflow is the supervisor loop. It asks `TaskSupervisor` to classify the incoming description and produce a `RoutingPlan`, then runs a routing guardrail before any subagent is contacted. If the plan passes, the workflow fans work out to `DataSubagent` and `SummarySubagent` in parallel. Both results return to the workflow, which asks `TaskSupervisor` to assemble the final `TaskResult`. Every lifecycle transition is written as a command to `TaskEntity`, whose events project into `TaskView`. The endpoint reads and streams the view; `RoutingEvalSampler` reads it to pick a completed task to score.

## Interaction sequence

The sequence diagram traces the primary journey: route, guardrail check, parallel fan-out, join, assemble, persist. The `par` block is the delegation-supervisor-workers core — both subagents run concurrently and the workflow joins their results before assembly. The guardrail step appears before the `par` block: a failing plan terminates the workflow without ever invoking a subagent, which is the defining property of a `before-agent-invocation` guardrail.

## State machine

`TaskRecord` moves `RECEIVED → ROUTING` as soon as the workflow starts. From `ROUTING` there are two branches: `BLOCKED` when the guardrail rejects the routing plan, and `IN_PROGRESS` when the plan is cleared. From `IN_PROGRESS` the task reaches one of three terminals: `COMPLETED` (assembly succeeded), `DEGRADED` (a subagent timed out), or — in principle — a re-block if the assembly itself is rejected (not wired in this sample). `COMPLETED` accepts one further `RoutingEvalScored` self-transition when `RoutingEvalSampler` records a score.

## Entity model

`TaskQueue` seeds one `TaskEntity` per submitted description. A task owns at most one `RoutingPlan`, one `DataBundle`, one `SummaryOutput`, and one `TaskResult`. The view row mirrors the entity record with `Optional<T>` on every field that is null before its transition (Lesson 6), so the UI can render partial state at each lifecycle step.
