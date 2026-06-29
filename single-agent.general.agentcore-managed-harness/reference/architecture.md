# Architecture — managed-harness-tool-use-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one tool-calling LLM loop. `QueryEndpoint` accepts a question submission, writes a `QuerySubmitted` event onto `QueryEntity`, and starts a `QueryWorkflow` instance. The workflow's `runStep` calls `OperationsAgent` — the single AutonomousAgent — with the question and the selected agent profile's tool declarations. The managed runtime then runs the tool-calling loop: the agent decides which tool to call next, the runtime fires `ToolAllowlistGuardrail` via the `before-tool-invocation` hook, and if the call is allowed, dispatches it to `ToolStubs`. The agent receives the tool's output and decides whether to call another tool or emit a final `QueryAnswer`. Once a `QueryAnswer` is returned, the workflow's `recordStep` writes it to `QueryEntity`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `ToolAllowlistGuardrail` and `ToolStubs` are supporting classes, not agents. `AgentProfiles` is a static registry. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note the inner loop between `OperationsAgent`, `ToolAllowlistGuardrail`, and `ToolStubs`: this is the managed tool-calling loop that the runtime executes on behalf of the agent. The agent drives it by deciding which tool to call next; the runtime enforces the guardrail and dispatches to the stub. The loop repeats until the agent emits a final prose answer or exhausts its iteration budget.

Two workflow steps bound the outer shape:

1. `runStep` — the entire tool-calling loop, bounded by a 90 s step timeout.
2. `recordStep` — a single entity write, bounded by a 10 s step timeout.

## State machine

Four states. The paths:

- The happy path is `SUBMITTED → RUNNING → ANSWERED`.
- Two failure transitions land in `FAILED`: a workflow error during `SUBMITTED` (startup failure), and an agent error or guardrail-exhaustion during `RUNNING`. A `FAILED` query's question is preserved on the entity — the UI shows the partial state so the user can inspect the submitted question and resubmit if needed.
- There is no `APPROVED` or `ACTED` state. The answer is advisory; the human reads it and acts outside the system.

## Entity model

`QueryEntity` is the source of truth. It emits four event types. `QueryView` projects every event into a row used by the UI. `QueryWorkflow` both reads (`getQuery`) and writes (`markRunning`, `recordAnswer`, `fail`) on the entity. The relationship between `OperationsAgent` and `QueryAnswer` is "returns" — the agent's task result is the answer record. The relationships between `OperationsAgent`, `ToolAllowlistGuardrail`, and `ToolStubs` are mediated by the managed runtime; they do not flow through the entity.

## Tool-allowlist governance flow

For any answer that lands in the entity log, every tool call in the trace passed through:

1. **ToolAllowlistGuardrail** — the tool name was in the agent's profile allowlist; arguments were not wildcards or empty.
2. **ToolStubs** — the call was dispatched to a deterministic in-process implementation that returns realistic-looking data.
3. **OperationsAgent** — the model received the tool output, reasoned over it, and decided whether to call another tool or emit a final answer.

Removing the guardrail opens the possibility that the agent calls a write-capable or cross-tenant tool. Replacing the stubs with real external service calls opens latency and availability risks; the guardrail remains the admission gate either way.
