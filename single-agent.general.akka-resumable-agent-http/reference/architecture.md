# Architecture — akka-resumable-agent-http

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call and one slow in-process tool. `AgentEndpoint` accepts a run request, writes `RunInitiated` onto `AgentRunEntity`, and starts `AgentRunWorkflow`. The workflow's `toolCallStep` calls `WeatherQueryAgent` — the single AutonomousAgent — with the location and query type as task instructions. Before `SlowWeatherTool` executes, the `before-tool-call` guardrail (`ToolCallGuardrail`) validates the agent's invocation arguments. Once the tool completes, the workflow writes `ToolCallCompleted` and advances to `reportStep`. On process restart, the workflow detects whether `AgentRunEntity.status` is `TOOL_CALLED` or `REPORTING`, emits `RunResumed`, and calls `IncidentEvaluator` in `incidentStep` before completing the run normally. `AgentRunView` projects every entity event into a read-model row; `AgentEndpoint` serves the read model over REST and SSE.

The graph has no second agent. `IncidentEvaluator` is deterministic and rule-based — not an LLM call. That is what makes this a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). The key moment is the 8-second gap during `SlowWeatherTool.call(...)` — this is the window where a developer can kill the process to trigger crash-resume (J2). The note in the diagram marks this point explicitly.

Two distinct pauses in the flow:

1. The `SlowWeatherTool` artificial delay — configurable, default 8 s. This is intentional; the delay makes the kill-window observable.
2. The LLM round-trip in `WeatherQueryAgent` — bounded by `toolCallStep`'s 120 s timeout.

The `incidentStep` runs only when `resumeCount > 0`; on a clean first run it is skipped.

## State machine

Seven states, including `RESUMED`. The interesting paths:

- The happy path is `INITIATED → TOOL_CALLED → REPORTING → COMPLETED`.
- A crash during `TOOL_CALLED` or `REPORTING` lands in `RESUMED`; on workflow restart it transitions back into the interrupted step — `TOOL_CALLED` if the tool had not yet finished, or `REPORTING` if the tool had already returned. The `RESUMED` state is transient: the workflow moves through it on the same restart.
- Two `FAILED` transitions: a guardrail-exhaustion during `TOOL_CALLED` (all 3 iterations rejected), and an unrecoverable error in `initStep`.
- There is no `APPROVED` state. The report is informational; no downstream enforcement action is taken by the system.

## Entity model

`AgentRunEntity` is the source of truth. It emits seven event types. `AgentRunView` projects every event into a row used by the UI. `AgentRunWorkflow` both reads (`getRun`) and writes (`initiate`, `startToolCall`, `completeToolCall`, `generateReport`, `complete`, `fail`, `resume`, `recordIncident`) on the entity. The relationship between `WeatherQueryAgent` and `WeatherReport` is "returns" — the agent's task result is the report record.

## Crash-resume governance flow

For any run that reaches `COMPLETED` after a crash, the sequence was:

1. **Workflow checkpoint** — Akka persisted `ToolCallCompleted` before the crash; on restart, `toolCallStep` is not re-invoked.
2. **Resume detection** — workflow `onInit` reads entity status and emits `RunResumed` if non-terminal.
3. **`before-tool-call` guardrail** — any NEW tool invocation after a retry still passes through `ToolCallGuardrail` before executing.
4. **On-incident evaluator** — `incidentStep` records a durable `IncidentEvent` classifying the crash by severity, elapsed time, and interrupted step name.

Each step is independent. Removing the incident evaluator silences the audit trail without affecting run completion.
