# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A task request enters through `CompositionEndpoint` (`POST /api/tasks`) or is dripped by `RequestSimulator` every 60 seconds. Either path writes a `TaskSubmitted` event onto `TaskRequestQueue`. `TaskRequestConsumer` subscribes to those events and starts one `CompositionWorkflow` per submission, keyed by `requestId`.

The workflow is the supervisor's execution harness. It calls `Supervisor` to produce a `RoutingDecision`, runs a before-tool-call guardrail to inspect that decision, then dispatches the selected sub-agent (`SummarizerAgent`, `ClassifierAgent`, or `TranslatorAgent`) and collects its `ToolResult`. It calls `Supervisor` a second time to assemble the final `TaskResult`. Every state transition is written as a command to `TaskRequestEntity`, whose events project into `CompositionView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a completed request to score.

## Interaction sequence

The sequence diagram traces the primary path: submit, route, guardrail check, dispatch to the selected sub-agent, assemble, and persist. The `alt` block at the guardrail captures the BLOCKED path. The single `SUB` participant represents whichever of the three sub-agents is selected — only one runs per request. This is the agents-as-tools contract: the supervisor's routing decision selects the tool, the workflow dispatches it, and the result flows back to the supervisor for assembly.

## State machine

`TaskRequest` moves `QUEUED → IN_PROGRESS`, then to one of three terminals: `COMPLETED` (guardrail passed and sub-agent returned), `BLOCKED` (guardrail rejected the tool call), or `TIMED_OUT` (sub-agent did not respond within 60 seconds). `COMPLETED` accepts one further `TaskEvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`TaskRequestQueue` seeds one `TaskRequest` per submission. A request owns at most one `RoutingDecision`, one `ToolResult`, and one `TaskResult`. The view row mirrors the request with `Optional<T>` on every field that is null before its transition (Lesson 6).
