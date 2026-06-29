# Architecture — ui-demo

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `CopilotEndpoint` accepts a prompt submission, writes a `TurnSubmitted` event onto `SessionEntity`, and starts a `SessionWorkflow` instance. The workflow's `agentCallStep` calls `CopilotAgent` — the single AutonomousAgent — with the user prompt and session history formatted as a conversation transcript. The agent's `before-agent-response` guardrail (`ResponseGuardrail`) validates each candidate response. Once a response passes, the workflow writes `TurnCompleted` and calls `markIdle`. `SessionView` projects every entity event into a read-model row; `CopilotEndpoint` serves the SSE stream from the agent's task output directly to the browser, and the read model to the UI over REST.

The graph deliberately has no second agent. Session history management, view projection, and the SSE relay are all deterministic in-process logic. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note the two-track output from the agent call:

1. **Streaming track** — token-chunk SSE envelopes flow from the agent's task output directly through `CopilotEndpoint` to the browser as the agent writes them. The chat bubble fills in real time.
2. **Commit track** — after the final response passes the guardrail, the workflow writes `TurnCompleted` to the entity, which in turn emits the `RESPONSE_COMPLETE` AG-UI envelope closing the stream.

The guardrail sits between the agent and the commit track only. Streaming tokens are not individually validated — the guardrail sees the complete structured payload once the agent finishes generating.

## State machine

The state machine tracks the session as a whole across multiple turns. A session begins `ACTIVE` on creation. Each `TurnSubmitted` event keeps it `ACTIVE` (the streaming is happening). A `TurnCompleted` event moves it to `IDLE` — ready for the next prompt. A `TurnFailed` event (agent error or guardrail exhaustion) lands the session in `FAILED`. An `IDLE` session accepts the next `TurnSubmitted`, cycling back to `ACTIVE`. This models a normal back-and-forth copilot conversation.

## Entity model

`SessionEntity` is the source of truth. It emits five event types. `SessionView` projects every event into a row used by the session list in the UI. `SessionWorkflow` both reads (`getSession`) and writes (`markStreaming`, `completeTurn`, `failTurn`, `markIdle`) on the entity. The relationship between `CopilotAgent` and `CopilotResponse` is "returns" — the agent's task result is the response record that gets committed.

## Guardrail governance

For any response that lands in the session history, the payload has passed:

1. **ResponseGuardrail — answer non-blank**: empty answers are caught before reaching the entity log.
2. **ResponseGuardrail — citation source non-empty**: citations with empty sources, which could not be verified by a user, are rejected.
3. **ResponseGuardrail — citation index positive**: malformed index values, which would break the citation-marker rendering in the UI, are rejected.
4. **ResponseGuardrail — token count ceiling**: over-length responses that would overflow the UI bubble or downstream context windows are rejected before they are streamed.

Each check is independent. A response that passes all four is committed; one that fails any triggers an agent retry within the 3-iteration budget. If all iterations are exhausted, the turn fails gracefully — the session remains addressable and the user can retry their prompt.
