# Architecture — custom-orchestration-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`TaskEndpoint` is the entry point for both task submissions and strategy management. A task submission writes a `TaskSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes and starts a `TaskWorkflow` per submission.

The workflow's first action is to load the currently active `OrchestrationStrategy` from `StrategyRegistry`. With the strategy in hand, it asks `OrchestratorAgent` to initialize an `ExecutionContext`. The loop then begins: on each tick the orchestrator produces a `RoutingDecision`; the workflow runs the decision through `ToolCallGuardrail`, dispatches the approved `ToolCall` to `ToolDispatcherAgent`, scrubs the result, and appends a `TraceEntry` to `TaskEntity`.

Every event emitted on `TaskEntity` is projected into `TaskView`, which the UI consumes via SSE. Two TimedActions run alongside: `RequestSimulator` drips sample tasks; `StuckTaskMonitor` marks tasks that stall in `EXECUTING` for more than five minutes.

`SystemControlEntity` is a single-instance entity keyed by `"global"`. Operators flip the halt flag from the UI; the workflow polls it before each route tick.

The critical design point: the workflow snapshots the strategy into its own state in `loadStrategyStep`. Swapping the active strategy for future tasks (via `POST /api/strategies/activate/{name}`) does not affect workflows already running — they complete on the strategy they started with.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the custom-strategy routing cycle: check-halt → route → guardrail → dispatch → sanitize → record → evaluate. The diagram shows a single `ToolDispatcherAgent` participant; in code the dispatcher routes internally to the matching fixture handler based on `toolName`.

## State machine

`TaskEntity` has six states. `INITIALIZING` covers the two setup steps (strategy load + context initialization). `EXECUTING` is the loop state; most events fire here without changing the status. Terminal states: `COMPLETED` (successful conclude), `FAILED` (abort signal or iteration budget exhausted), `HALTED` (operator halt), `STUCK` (no progress for five minutes).

## Entity model

`TaskEntity` is the source of truth for the task lifecycle and execution trace. `StrategyRegistry` is the source of truth for named strategies and the active-strategy pointer. `SystemControlEntity` carries the operator halt flag. `RequestQueue` is the submission audit log. `TaskView` is the only read-side projection; the UI never queries entities directly.

## Concurrency & timeouts

- Per-step timeouts: `initContextStep` 45 s, `routeStep` 45 s, `dispatchStep` 90 s, `concludeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(TaskWorkflow::error)`.
- Iteration budget: capped by `strategy.maxRouteIterations`; exceeding it fails the task with reason `"iteration budget exhausted"`.
- Revisit budget: capped by `strategy.maxRevisits`; the orchestrator cannot exceed it regardless of how many `Revisit` decisions it produces.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
- Strategy isolation: the workflow holds a snapshot of the strategy; strategy swaps are non-disruptive to in-flight tasks.
- Stuck detection: `StuckTaskMonitor` every 30 s; tasks `EXECUTING` for > 5 minutes are marked `STUCK`.
