# Architecture — durable-weather-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one durable agent invocation. `JobEndpoint` accepts a query submission, writes a `JobQueued` event onto `WeatherJobEntity`, and then starts an `AgentWorkflow` instance — either via the `triggerAgent` entry transition (fire-and-wait) or the `callAgent` entry transition (child-workflow mode). The workflow's `runAgentStep` calls `WeatherAgent` — the single AutonomousAgent — passing the user's query text as `TaskDef.instructions(...)`. The agent drives three tool calls (`get_current_weather`, `get_forecast`, `get_weather_alerts`) through `WeatherToolActivity`. Before each HTTP call, `ToolCallGuardrail` inspects the tool's arguments; a passing call proceeds to the fixture (or real) HTTP layer, a failing call returns a structured rejection to the agent loop. Once all tool calls complete, the agent assembles the `WeatherReport` and returns it to the workflow. `AgentWorkflow.finalizeStep` calls `WeatherJobEntity.complete(report)`. `JobView` projects every entity event into a read-model row; `JobEndpoint` serves that read model to the UI over REST and SSE.

The graph deliberately has no second agent. `ToolCallGuardrail` is a pure logic class — no LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the `trigger_agent` happy path (J1). Key pauses in the flow:

1. The `startAgentStep` is fast — it writes `JobStarted` and advances immediately.
2. The `runAgentStep` drives multiple sequential tool calls through the guardrail and activity layer. Each activity is a durable checkpoint: if the process restarts between activities, only activities not yet completed are replayed.
3. The guardrail runs synchronously before each HTTP call. A blocked call adds one iteration to the agent's budget and a `BLOCKED` row to the tool-call log; a passed call consumes no iteration — only the LLM turn that decides to call the tool counts.

The `call_agent` path differs in how the workflow is started: `JobEndpoint` calls `asyncCall(AgentWorkflow::callAgent, childJobId)` as a step inside the parent workflow, and the parent step suspends until the child workflow's `done` transition fires. Both the parent and child job IDs are independent `WeatherJobEntity` instances in the entity store.

## State machine

Four states. The paths:

- The happy path is `QUEUED → RUNNING → COMPLETED`.
- One failure transition lands in `FAILED`: an agent error, a guardrail budget exhaustion, or a `runAgentStep` timeout after 2 retries. A `FAILED` job's `WeatherJobEntity` preserves the partial `query` field for debugging.
- There is no partial-report state: the `WeatherReport` is written atomically as a single `JobCompleted` event. A job is either fully reported or not reported.

## Entity model

`WeatherJobEntity` is the source of truth. It emits four event types. `JobView` projects every event into a row used by the UI. `AgentWorkflow` both reads (to check status) and writes (`start`, `complete`, `fail`) on the entity. The relationship between `WeatherAgent` and `WeatherReport` is "returns" — the agent's task result is the report record.

## Governance flow

For any report that lands in the entity log, the tool calls passed through:

1. **before-tool-call guardrail** — argument validation runs before any HTTP call exits the process. The agent never reaches the network with a blocked argument set.
2. **Activity durability** — the workflow activity layer ensures that a tool call's result is stored in the journal before the agent sees it. A crash between activities does not cause a tool to be called twice.
3. **Iteration budget** — `maxIterationsPerTask(4)` caps the number of times the agent can rephrase a blocked tool call. If the budget is exhausted the workflow fails cleanly with an error record rather than running indefinitely.

Each of these layers is independent. Removing the guardrail opens the HTTP boundary; removing the activity layer means a crash mid-call leaves the job in an unknown state; removing the iteration budget means a pathological query could drive the agent in a loop until the `runAgentStep` 120 s timeout fires.
