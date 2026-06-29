# Architecture — code-agent-chat-ui

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ChatEndpoint` accepts a new message, writes a `UserMessageReceived` event onto `ChatSessionEntity`, and signals `ChatSessionWorkflow`. The workflow's `activeStep` polls the entity until the new message is present, then calls `CodeAssistantAgent` — the single AutonomousAgent — with the conversation history as `TaskDef.instructions(...)`. The agent runs `WebSearchTool` and `CodeExecutionTool` in sequence, replanning every 3 steps. Two guardrails bracket the agent: `ToolCallPolicy` fires before each tool invocation (before-tool-call hook) and `ResponseGuardrail` fires before the final response reaches the stream (before-agent-response hook). Once a response passes both checks, the workflow writes `AssistantMessageRecorded` onto the entity. `ChatView` projects every entity event into a read-model row; `ChatEndpoint` serves the read model to the UI over REST and SSE.

The `ToolCallValidator` Consumer handles a parallel audit path: every `ToolCallRequested` event emitted by the entity flows through the Consumer, which records the approval or block decision as `ToolCallResolved`. This keeps the tool-call audit log consistent regardless of which iteration the agent is on.

The graph deliberately has no second agent. `ResponseGuardrail` and `ToolCallPolicy` are supporting classes wired into the agent's guardrail-configuration block — not agents. That is what makes this blueprint a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system waits on something:

1. The `activeStep` polling loop inside the workflow — polls `ChatSessionEntity` every 1 s up to its 90 s timeout, advancing as soon as the user message is present.
2. The agent task itself — bounded by `activeStep`'s 90 s timeout, which accommodates multi-tool latency and plan-revision overhead.

Each tool call triggers a `ToolCallRequested` event on the entity and a `ToolCallResolved` event after the `ToolCallValidator` Consumer processes it. The two events bracket each tool invocation in the audit log.

## State machine

Four states. The interesting paths:

- The happy path is `INITIALIZING → ACTIVE → IDLE` (after a period of inactivity) or `ACTIVE → ACTIVE` for continued conversation.
- A session that receives a new user message while in `IDLE` transitions back to `ACTIVE`, re-activating the workflow.
- Two failure transitions land in `FAILED`: a guardrail-exhaustion (all 12 iterations fail validation) or a workflow error step that cannot recover. A `FAILED` session's prior message history is preserved on the entity — the UI shows the history with a failure badge on the last turn.
- There is no `COMPLETED` terminal state. Sessions remain in `IDLE` between turns, ready for the next message.

## Entity model

`ChatSessionEntity` is the source of truth. It emits seven event types. `ChatView` projects every event into a row used by the UI. `ToolCallValidator` subscribes to `ToolCallRequested` events to run the approval gate. `ChatSessionWorkflow` both reads (`getSession`) and writes (`markActive`, `markIdle`, `recordResponse`, `recordPlanRevision`, `fail`) on the entity. The relationship between `CodeAssistantAgent` and `ChatResponse` is "returns" — the agent's task result is the response record.

## Defence-in-depth governance flow

For any agent response that lands in the entity log, the request passed through:

1. **ToolCallPolicy** (before-tool-call hook) — each tool call checked before execution; blocked calls never reach the tool.
2. **CodeAssistantAgent** — one model call, one structured output.
3. **ResponseGuardrail** (before-agent-response hook) — empty, malformed, or leaking responses are caught before the stream.
4. **ToolCallValidator** Consumer — every tool invocation recorded and resolved in the entity's audit log, independent of the guardrail path.

Each check is independent. Removing one opens an explicit gap the others do not cover.
