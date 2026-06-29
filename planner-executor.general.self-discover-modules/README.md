# Akka Sample: Self-Discover Agent

A StructurerAgent selects and adapts a reasoning module set for a given task, composes those modules into an actionable reasoning structure, then hands that structure to a SolverAgent that executes each module step and produces a final answer. Demonstrates the **planner-executor** coordination pattern with a single governance eval hook on the composed structure.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The module library and task corpus are seeded fixtures bundled with the service.

## Generate the system

```sh
cp -r ./planner-executor.general.self-discover-modules  ~/my-projects/self-discover-modules
cd ~/my-projects/self-discover-modules
```

(Optional) Edit `SPEC.md` to change the module library, model provider, or the quality threshold used by the eval hook.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **StructurerAgent** — AutonomousAgent that selects relevant modules from the module library, adapts their descriptions to the task, and composes them into an ordered `ReasoningStructure`.
- **SolverAgent** — AutonomousAgent that executes each module step in the composed structure and produces a `TaskAnswer`.
- **TaskWorkflow** — Workflow with a structure → eval → solve → record loop, plus a revise branch when the eval rejects the structure, and terminal exit states.
- **TaskEntity** — EventSourcedEntity holding the task lifecycle, the composed structure, per-step execution records, and the final answer.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **TaskView** — projection used by the UI.
- **TaskRequestConsumer** — Consumer that starts a workflow per submitted task.
- **RequestSimulator** — TimedAction that drips sample tasks every 90 s so the UI is never empty.
- **StuckTaskMonitor** — TimedAction that marks stalled tasks `STUCK` after 5 minutes.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust `ReasoningStructure` fields (e.g., add a `confidence` weight per module).
- `prompts/structurer.md` — narrow the module selection to a specific domain (e.g., scientific reasoning only).
- `eval-matrix.yaml` — raise the eval quality threshold or add a second eval control for solver output.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → structurer selects modules, eval passes, solver executes, task completes within ~3 minutes.
2. Submit a task that produces a low-quality structure → eval hook fires, structurer revises, revised structure passes, task completes.
3. Submit a task and click **Halt new dispatches** while it is `EXECUTING` → in-flight module step finishes; task moves to `HALTED`.
4. Structurer exhausts the revise budget → task ends in `FAILED` with a clear `failureReason`.

## License

Apache 2.0.
