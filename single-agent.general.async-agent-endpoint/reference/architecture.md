# Architecture — async-agent-endpoint

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `RunEndpoint` accepts a submission, writes a `RunSubmitted` event onto `AgentRunEntity`, and immediately starts an `AgentRunWorkflow` instance. The workflow's `runStep` calls `CodeRunnerAgent` — the single AutonomousAgent — with the task prompt as `TaskDef.instructions(...)`. Before any tool call executes, the agent's `before-tool-call` guardrail (`SandboxGuardrail`) inspects the generated code for forbidden constructs. Once a compliant tool call passes, the code runs, and the agent returns a `RunResult`. The workflow writes `RunCompleted` and the entity transitions to `COMPLETED`. `RunView` projects every entity event into a read-model row; `RunEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `SandboxGuardrail` is a policy-check class, not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two moments worth noting:

1. The `RunEndpoint` starts the workflow immediately after submitting to the entity — there is no Consumer subscription lag because the workflow is started directly by the endpoint, not triggered by an event.
2. The `before-tool-call` guardrail intercepts between the agent's decision to call a tool and the tool's execution. On the happy path this is a sub-millisecond pass-through; on the J2 path the rejection is returned to the agent loop, which retries with a rewritten code generation.

The agent call is bounded by `runStep`'s 60 s timeout. The guardrail check is synchronous and adds no measurable latency on the pass path.

## State machine

Four states. The notable paths:

- The happy path is `SUBMITTED → RUNNING → COMPLETED`.
- One failure transition: an agent error or guardrail exhaustion during `RUNNING` lands in `FAILED`. The `RunFailed` event preserves the `FailureKind` and message for the UI.
- There is no intermediate state between `SUBMITTED` and `RUNNING` — the workflow starts synchronously from the endpoint, so the transition is nearly instantaneous.

## Entity model

`AgentRunEntity` is the source of truth. It emits four event types. `RunView` projects every event into a row used by the UI. `AgentRunWorkflow` both reads (via `getRun` if needed) and writes (`markRunning`, `recordCompleted`, `recordFailed`) on the entity. The relationship between `CodeRunnerAgent` and `RunResult` is "returns" — the agent's task result is the run-result record.

## Concurrency and event-loop behaviour

The key architectural property of this blueprint is that the agent's blocking work (LLM inference + code execution) never occupies the Akka event loop directly. The Akka runtime dispatches the workflow step on a thread pool; within that step, the agent call is a standard async component invocation. Concurrent submissions each get their own workflow instance with their own agent instance id (`"runner-" + runId`), so runs do not serialize. J4 confirms this: two runs submitted in rapid succession both transition to `COMPLETED` with overlapping timestamps.
