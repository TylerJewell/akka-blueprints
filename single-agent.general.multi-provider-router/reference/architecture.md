# Architecture — multi-provider-router

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call with a load-balancing router in front of it. `RouterEndpoint` accepts a prompt submission, writes a `CallSubmitted` event onto `CallEntity`, and starts a `RoutingWorkflow` instance. The workflow's `dispatchStep` calls `RouterAgent` — the single AutonomousAgent — with the prompt text as `TaskDef.instructions(...)`. The agent's underlying `LiteLLMRouterModel` selects one of the two configured backends (OpenAI `gpt-4o-mini` or Anthropic Claude) using simple-shuffle ordering and completes the call. The agent returns a `CallResult`; the workflow writes `CallCompleted` to `CallEntity`. The `PerformanceMonitor` Consumer subscribes to `CallCompleted` events; when the rolling window fills (every 5 calls), it invokes `EvaluationAggregator` to compute per-provider statistics and writes the result to `MonitorView`. `CallView` projects every entity event into a call-record row; `RouterEndpoint` serves both views over REST and SSE.

The graph has no second agent. `EvaluationAggregator` is a deterministic rule-based aggregator — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model, even though that one component routes across two provider backends.

## Interaction sequence

The sequence traces the happy path (J1). Note the two distinct coordination points:

1. `RouterEndpoint` starts the `RoutingWorkflow` immediately after minting the callId — the workflow owns the agent call from that point.
2. The `PerformanceMonitor`'s window-threshold check is asynchronous; it runs after the `CallCompleted` event lands and does not block the call's response to the user.

The agent call itself is bounded by `dispatchStep`'s 60 s timeout, which accommodates provider latency variance. The evaluation aggregation runs in-process inside the Consumer and finishes in milliseconds — no external service, no LLM call.

## State machine

Four states. The paths:

- The happy path is `PENDING → DISPATCHED → COMPLETED`.
- Two failure transitions land in `FAILED`: a workflow-start error during `PENDING`, and a provider error or timeout during `DISPATCHED`. A `FAILED` call's partial data is preserved on the entity — the UI shows the error reason for operators.
- There is no `APPROVED` or `REVIEWED` state. The call's response is returned directly; the system is informational, not advisory.

## Entity model

`CallEntity` is the source of truth. It emits four event types. `CallView` projects every event into a call row used by the UI. `PerformanceMonitor` subscribes to entity events to feed the rolling quality window. `RoutingWorkflow` both reads (implicitly, via the workflow's awareness of prior steps) and writes (`dispatch`, `complete`, `fail`) on the entity. The relationship between `RouterAgent` and `CallResult` is "returns" — the agent's task result is the call-result record.

## Provider-routing and monitoring flow

For any call that lands in `COMPLETED`:

1. **LiteLLMRouterModel shuffle** — the router selects the backend before the agent prompt is sent; the selection is invisible to the agent's instructions.
2. **RouterAgent** — one model call, one structured output (`CallResult` with `selectedProvider` populated).
3. **PerformanceMonitor window** — every 5th completed call triggers aggregation; P50/P95 latency and quality score per provider are surfaced on the Eval Matrix tab.

The monitor is advisory. If a provider's P95 latency exceeds a threshold or its quality score drops below 3, the Eval Matrix tab flags the provider in amber — but individual calls continue routing normally until an operator intervenes.
