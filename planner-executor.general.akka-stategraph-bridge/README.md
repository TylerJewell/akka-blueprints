# Akka Sample: Graph API Plugin

A `GraphRunner` executes a user-defined `StateGraph` — nodes connected by typed edges, including conditional branches and back-edges (cycles) — durably on Akka. Each node invocation is a traceable step; the graph's mutable state is a typed `GraphState` record held by an event-sourced entity. A before-tool-call guardrail vets every node that bears a tool annotation before the node runs.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Node tool calls are simulated with seeded fixtures; no external services are contacted.

## Generate the system

```sh
cp -r ./planner-executor.general.akka-stategraph-bridge  ~/my-projects/akka-stategraph-bridge
cd ~/my-projects/akka-stategraph-bridge
```

(Optional) Edit `SPEC.md` to change the graph definition, the node agent prompts, or the guardrail allow-list.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that reads a raw graph definition and produces a validated `GraphPlan` (nodes, edges, entry point, terminal nodes, cycle annotations).
- **NodeAgent** — AutonomousAgent that executes a single graph node: calls any attached tool fixture, produces a `NodeOutput`, and updates the `GraphState` fields the node owns.
- **EdgeRouterAgent** — AutonomousAgent that evaluates a conditional edge's predicate against the current `GraphState` and returns a `RoutingDecision` (next node id or terminal marker).
- **GraphWorkflow** — Workflow driving the plan → execute-node → route → record-state → check-cycle → check-halt loop with replan and terminal branches.
- **GraphRunEntity** — EventSourcedEntity holding the graph run lifecycle, the current `GraphState`, the execution trace, and the final result.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag (single instance keyed by `"global"`).
- **RunQueueEntity** — EventSourcedEntity acting as audit log for submitted graph run requests.
- **GraphRunView** — View projection of `GraphRunEntity` events for the UI list.
- **RunRequestConsumer** — Consumer subscribed to `RunQueueEntity` events; starts a `GraphWorkflow` per submission.
- **RunSimulator** — TimedAction that drips a sample graph definition every 90 s.
- **StuckRunMonitor** — TimedAction that marks runs stuck in `EXECUTING` past 5 minutes.
- **GraphEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample graph definitions the simulator drips, or disable the simulator.
- `SPEC.md §5` — adjust the `GraphState` fields (e.g., add a `confidenceScore` or domain-specific accumulator).
- `prompts/planner.md` — narrow the planner to a specific graph topology (e.g., DAG-only, no back-edges).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-node quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a graph definition → runner plans, executes all nodes in order, follows conditional edges, completes within ~3 minutes.
2. Submit a graph with a tool-bearing node whose tool annotation is outside the allow-list → guardrail blocks the node execution; runner records the block and the planner either replans or the run ends in `FAILED`.
3. Submit a graph and click **Halt new dispatches** while a run is `EXECUTING` → in-flight node finishes; no further node dispatches; run moves to `HALTED`.
4. A node returns output containing a secret-shaped string → sanitizer scrubs it before the state update lands in `GraphRunEntity`; the EdgeRouterAgent's next prompt never contains the literal secret.

## License

Apache 2.0.
