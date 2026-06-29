# Architecture — function-calling-agent-baseline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call in a tool-calling loop. `AgentRunEndpoint` accepts a query submission, writes a `RunSubmitted` event onto `AgentRunEntity`, then immediately starts an `AgentRunWorkflow`. The workflow's `startStep` transitions the entity to `RUNNING`, then `runStep` calls `FunctionCallingAgent` — the single AutonomousAgent — with the query and tool definitions as the task instruction text. The agent enters a function-calling loop: it proposes tool invocations, each of which passes through `ToolCallGuardrail` before execution; accepted calls are dispatched to `InProcessToolExecutor`, whose results are fed back to the agent. When the agent decides it has enough information, it proposes a final `AgentAnswer`. That answer passes through `AnswerGuardrail` before leaving the agent's loop. Once the answer passes, the workflow writes `AnswerRecorded` to the entity. `AgentRunView` projects every entity event into a read-model row; `AgentRunEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `InProcessToolExecutor` is plain Java — no LLM call — and both guardrail classes are validation logic. This is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct moments where control passes between components:

1. The endpoint → entity → workflow handoff, which happens synchronously before the response returns to the UI. The UI receives the `runId` and begins listening on SSE.
2. The tool-calling loop inside `FunctionCallingAgent` — each iteration sends a call through `ToolCallGuardrail`, executes the tool if accepted, and sends results back to the agent for the next iteration.
3. The final answer through `AnswerGuardrail` — one additional validation cut before the answer is committed to the entity.

The `runStep` is bounded by a 120 s timeout to accommodate multi-round loops (Lesson 4). `InProcessToolExecutor` is synchronous and adds negligible latency.

## State machine

Four states. The paths:

- The happy path is `SUBMITTED → RUNNING → ANSWER_RECORDED`.
- Two failure transitions land in `FAILED`: a workflow-level error during `SUBMITTED` (e.g., workflow fails to start), and an agent error or iteration-budget exhaustion during `RUNNING`. A `FAILED` run's prior data is preserved on the entity; the UI shows the partial state.
- There is no `REVIEWED` or `APPROVED` state. The answer is the terminal output; what the caller does with it is outside the system. The blueprint stops at `ANSWER_RECORDED`.

## Entity model

`AgentRunEntity` is the source of truth. It emits four event types. `AgentRunView` projects every event into a row used by the UI. `AgentRunWorkflow` both reads (`getRun`) and writes (`markRunning`, `recordAnswer`, `fail`) on the entity. The relationship between `FunctionCallingAgent` and `AgentAnswer` is "returns" — the agent's task result is the answer record, which includes the full tool-call trace.

## Two-guardrail governance

For any answer that lands in the entity log, the query passed through:

1. **ToolCallGuardrail (before-tool-call)** — every tool invocation is checked before it fires. Invalid tool names, missing parameters, and wrong-type values are caught and returned to the agent as structured rejections.
2. **FunctionCallingAgent** — one model, one tool-calling loop, one structured output per run.
3. **AnswerGuardrail (before-agent-response)** — the final answer is checked for structure, completeness, and credential-like content before it leaves the agent's loop.

The two guardrails are at different cut points: one stops bad inputs from reaching the tool executor; the other stops problematic outputs from reaching the caller. Neither silently covers the other's gap.
