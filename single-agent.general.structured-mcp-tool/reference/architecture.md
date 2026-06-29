# Architecture — structured-mcp-tool

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a submission, writes a `QuerySubmitted` event onto `QueryEntity`, and immediately starts a `WeatherQueryWorkflow` instance. The workflow's `runStep` calls `WeatherQueryAgent` — the single AutonomousAgent — with the location and unit preference as task instructions. The agent issues one `get_weather_info` tool call to `InlineMcpServer`, which returns a `WeatherPayload`. The `after-tool-call` guardrail (`McpToolResultGuardrail`) validates the payload on its way back into the agent. If the payload passes, the agent composes a `WeatherSummary` and returns it to the workflow. The workflow writes `SummaryRecorded` onto `QueryEntity`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. `McpToolResultGuardrail` is a hook class, not an LLM — its checks are pure Java assertions on the payload's fields. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two points of interest:

1. The guardrail runs between the MCP tool response and the agent's reasoning step. In the happy path this is sub-millisecond; in the failure path (J2, J3) the rejection returns to the agent loop, which retries the tool call on the next iteration.
2. The `runStep` timeout is 60 s — long enough to accommodate LLM latency plus up to two tool-call retries.

## State machine

Four states. The paths:

- The happy path is `SUBMITTED → RUNNING → COMPLETED`.
- Two failure transitions land in `FAILED`: a workflow-start error during `SUBMITTED`, and an agent error or guardrail-exhaustion during `RUNNING`.
- `COMPLETED` and `FAILED` are both terminal. The query's data is preserved on the entity — the UI shows the partial state for the operator.

## Entity model

`QueryEntity` is the source of truth. It emits four event types. `QueryView` projects every event into a row used by the UI. `WeatherQueryWorkflow` both reads (`getQuery`) and writes (`markRunning`, `recordSummary`, `fail`) on the entity. `WeatherQueryAgent`'s relationship to `WeatherSummary` is "returns" — the agent's task result is the summary record. `InlineMcpServer` exposes `WeatherPayload` — the tool's raw structured output that the guardrail validates.

## Governance flow

For any summary that lands in the entity log, the tool response passed through:

1. **`get_weather_info` tool call** — the agent requests data from the inline MCP server.
2. **`after-tool-call` guardrail** — bad schema, out-of-range values, and unknown enum members are caught before the response enters the agent's reasoning context.
3. **`WeatherQueryAgent` reasoning** — one model call, one structured output.

Each step is independent. The guardrail fires regardless of which location is queried or which mock-response entry the `InlineMcpServer` happens to return. Removing it opens an explicit window where a schema-invalid payload would propagate silently into the summary.
