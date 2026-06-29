# Akka Sample: Custom Orchestration Agent

An Orchestrator runs under a pluggable orchestration strategy you supply. Instead of a fixed plan-dispatch-decide loop, the strategy object decides at each step how to route control — selecting which tool or sub-agent runs next, whether to revisit a prior result, or when to conclude. Demonstrates the **planner-executor** coordination pattern with a swappable strategy hook and embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Tools called by the orchestrator — search, file inspection, code generation, command execution — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.general.custom-orchestrator  ~/my-projects/custom-orchestration-agent
cd ~/my-projects/custom-orchestration-agent
```

(Optional) Edit `SPEC.md` to change the default strategy, the tool set, or any governance threshold.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OrchestratorAgent** — AutonomousAgent that holds the execution context and delegates routing decisions to the active `OrchestrationStrategy`.
- **StrategyRegistry** — EventSourcedEntity holding the named strategy configurations an operator can swap at runtime.
- **ToolDispatcherAgent** — AutonomousAgent that receives a `ToolCall` from the orchestrator and routes it to the correct simulated tool handler.
- **TaskWorkflow** — Workflow that drives the strategy-gated loop: initialize → route → dispatch → record → evaluate → route (again) or conclude.
- **TaskEntity** — EventSourcedEntity holding task lifecycle, the execution trace, and the final result.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag (single instance keyed by literal `"global"`).
- **TaskView** — projection used by the UI.
- **TaskRequestConsumer** — Consumer that starts a `TaskWorkflow` per submitted task.
- **RequestSimulator** — TimedAction dripping sample tasks every 90 seconds.
- **StuckTaskMonitor** — TimedAction marking stalled tasks.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the default strategy or add a second strategy variant.
- `SPEC.md §5` — adjust the `Task` record fields or add a `confidenceThreshold` to `RoutingDecision`.
- `prompts/orchestrator.md` — narrow the orchestrator to a single domain or change the routing heuristic.
- `eval-matrix.yaml` — add a `before-tool-call` control for per-tool policy if your tool set is broader.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → the orchestrator routes through multiple tools under the active strategy and completes within ~3 minutes.
2. Swap the active strategy at runtime → subsequent routing decisions follow the new strategy without restarting the service.
3. Trigger the operator halt → no new tool dispatches occur; the in-flight call finishes; the task moves to `HALTED`.
4. A tool result containing a secret-shaped string is scrubbed before it reaches the orchestrator's next routing prompt.

## License

Apache 2.0.
