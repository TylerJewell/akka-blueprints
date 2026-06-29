# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A question enters through `QueryEndpoint` (`POST /api/query`) or is dripped by `QuestionSimulator` every 60 seconds. Either path writes a `QuestionSubmitted` event onto `IndexCallQueue`. `IndexCallConsumer` subscribes to those events and starts one `QueryOrchestrationWorkflow` per submission, keyed by `sessionId`.

The workflow is the supervisor. It asks `QuerySupervisor` to decompose the question into sub-questions, fans each sub-question out to a separate `IndexWorker` instance in parallel, then asks `QuerySupervisor` to synthesise the collected results. Every transition is written as a command to `QuerySessionEntity`, whose events project into `QuerySessionView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a session to score.

The before-tool-call guardrail is embedded inside `IndexWorker`: before the seeded index tool executes, the hook validates the sub-question for relevance and safety. A rejected sub-question propagates back to the workflow as a `GuardrailException`, which records an `IndexCallRejected` event without stopping the remaining workers.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out to N index workers (each with a before-tool-call check), join, synthesise, persist. The `par` block is the delegation-supervisor-workers core. The number of parallel branches equals the number of sub-questions in the `DecompositionPlan`.

## State machine

`QuerySession` moves `DECOMPOSING → RETRIEVING`, then to one of three terminals: `SYNTHESISED` (all results collected and synthesised), `PARTIAL` (one or more workers timed out — synthesised from what arrived), or `BLOCKED` (every sub-question was rejected by the guardrail). `SYNTHESISED` accepts one further `SynthesisEvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`IndexCallQueue` seeds one `QuerySession` per question. A session owns one `DecompositionPlan`, a growing list of `IndexResult` objects (one per completed worker), and at most one `CombinedAnswer`. The view row mirrors the session with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
