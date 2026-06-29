# Architecture — akka-bridge

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `RunEndpoint` accepts a submission, writes `RunAccepted` onto `RunEntity`, and starts a `RunWorkflow`. The workflow's `agentStep` calls `BridgeAgent` — the single AutonomousAgent — with the task description as task instructions and the permitted tool definitions registered on the agent's definition. Before any tool call leaves the agent loop, `ToolCallGuardrail` fires its `before-tool-call` hook: it validates the tool name against the run's permitted-tools list, checks argument-schema conformance, and enforces the call budget. A permitted call is recorded on the entity and delivered to `AgentFrameworkAdapter` via a `ToolCallPermitted` event. The adapter executes the call in-process (via `MockToolExecutor`) and writes the result back via `RunEntity.recordToolResult`. When the agent finishes and returns a `RunResult`, the workflow writes `RunCompleted`. `RunView` projects every entity event into a read-model row; `RunEndpoint` serves the read model to the UI over REST and SSE.

There is no second agent. `MockToolExecutor` is plain Java; `AgentFrameworkAdapter` is a Consumer. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments are worth noting:

1. The guardrail check is synchronous — it fires inside the agent loop before the tool call is dispatched. The agent waits for the permit/block decision before proceeding.
2. Tool execution via `AgentFrameworkAdapter` is event-driven — the Consumer subscribes to `ToolCallPermitted` events and executes asynchronously. The agent loop receives the result via the task's tool-result callback, not as a direct return.

The `agentStep`'s 120 s timeout accommodates multi-turn execution with several sequential tool calls. The `completionStep` is a short bookkeeping write — 10 s is generous.

## State machine

Five states. The notable paths:

- The happy path is `ACCEPTED → RUNNING → COMPLETED`.
- `BLOCKED` is a distinct terminal state that means the guardrail halted the run cleanly (budget exhausted or the agent could not route around blocked tools). The partial tool-call log is preserved; the operator can inspect which calls succeeded before the block.
- `FAILED` covers agent errors, workflow timeouts, and infrastructure faults. The partial log is also preserved.
- There is no `APPROVED` state. `BridgeAgent` executes fully autonomously; the operator monitors via the audit log on `RunEntity`, not via an approval gate.

## Entity model

`RunEntity` is the source of truth. It emits nine event types covering run lifecycle and every tool-call state transition. `RunView` projects all events into a row for the UI. `AgentFrameworkAdapter` subscribes to `ToolCallPermitted` to trigger execution. `RunWorkflow` both reads (`getRun`) and writes (`start`, `complete`, `block`, `fail`) on the entity. `ToolCallGuardrail` writes `requestToolCall` and `permitToolCall` (or `blockToolCall`) synchronously inside the before-tool-call hook.

## Governance flow

For any tool call that reaches `AgentFrameworkAdapter`, the call has passed through:

1. **before-tool-call guardrail** — tool name is permitted, argument schema is valid, budget is not exhausted. The call is recorded with verdict `PERMITTED` before execution.
2. **MockToolExecutor** — in-process execution with seeded plausible results (real deployers replace this with an HTTP call to the actual framework).

For any tool call that does not reach the adapter:

1. The guardrail recorded `BLOCKED` with a rejection reason on the entity.
2. The agent loop received a tool error and adapted (or halted if budget was exhausted).

The audit log on `RunEntity` captures both paths. A tool call that is neither `PERMITTED` nor `BLOCKED` in the log is evidence of a guardrail bypass — which should never happen because the guardrail fires synchronously inside the agent loop before dispatch.
