# Architecture — multi-tool-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call and three outbound tool adapters. `SessionEndpoint` accepts a submission, writes a `SessionSubmitted` event onto `SessionEntity`, and starts a `SessionWorkflow` instance. The workflow's `validateStep` confirms the request is non-empty, then `dispatchStep` calls `ToolCallingAgent` — the single AutonomousAgent — with the user's request text as `TaskDef.instructions(...)`. The agent calls `WeatherTool`, `CurrencyTool`, and `UnitTool` in whatever combination the request requires. Before each tool call, the `before-tool-call` guardrail (`ToolCallGuardrail`) validates the input arguments; an invalid call is rejected and the agent reformulates. Once the agent produces a `ToolResponse`, the workflow writes `SessionAnswered`. `SessionView` projects every entity event into a read-model row; `SessionEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The three tool adapters are `@tool`-annotated helper classes, not AutonomousAgents. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1) for a request that requires two tool calls. Note the guardrail fire before each adapter call:

1. The `validateStep` is fast — a null-check and length guard, sub-second in normal operation.
2. The `dispatchStep` drives the agent call, which may involve several sequential tool calls. Each tool call triggers the `before-tool-call` guardrail. The step timeout is 120 s to accommodate LLM latency plus multiple tool round-trips.
3. Tool adapters are synchronous and in-process in this baseline. A real deployer swaps them for HTTP clients; the guardrail pattern is identical.

## State machine

Four states. The interesting paths:

- The happy path is `SUBMITTED → DISPATCHING → ANSWERED`.
- Two failure transitions land in `FAILED`: a validation error during `SUBMITTED` (e.g., empty request), and an agent error or iteration-budget exhaustion during `DISPATCHING`. A `FAILED` session's tool-call history is preserved on the entity — the UI shows the partial log so the user can diagnose which call failed.
- There is no `APPROVED` or `PROCESSED` state. The agent's answer is returned directly; the user reads it. The blueprint deliberately stops at `ANSWERED`.

## Entity model

`SessionEntity` is the source of truth. It emits five event types. `SessionView` projects every event into a row used by the UI. `SessionWorkflow` both reads (`getSession`) and writes (`startDispatch`, `recordAnswer`, `fail`) on the entity. The relationship between `ToolCallingAgent` and `ToolResponse` is "returns" — the agent's task result is the response record. The three tool adapters are called by the agent and are not Akka components; they have no direct relationship with the entity.

## Guardrail placement

The `before-tool-call` guardrail fires at a different cut than the common `before-agent-response` pattern. Rather than inspecting what the agent is about to say, it inspects what the agent is about to do — call an external service with a specific set of arguments. This placement matters when:

1. An argument out of range causes the external API to return a 4xx or charge a fee.
2. The agent infers a plausible-but-wrong parameter (e.g., currency code `EURO` instead of `EUR`) that would silently return wrong data if passed through.

The guardrail intercepts both cases before the adapter fires, gives the agent a structured rejection message, and lets it reformulate within the same task — all without the user ever seeing a failed state.
