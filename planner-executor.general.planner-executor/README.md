# Akka Sample: PlanAgents

A Planner agent breaks any submitted goal into a dynamic step-by-step plan; an Executor agent consumes each step, runs it, and feeds the result back so the Planner can revise when steps fail or produce unexpected output. Demonstrates the **planner-executor** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The executor surface is simulated inside the same Akka service using seeded fixtures; no external runtime is required.

## Generate the system

```sh
cp -r ./planner-executor.general.planner-executor  ~/my-projects/planner-executor
cd ~/my-projects/planner-executor
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that maintains a plan ledger (goal, open steps, completed steps, blocked steps) and produces revised plans on replanning requests.
- **ExecutorAgent** — AutonomousAgent that receives one step at a time, executes it against seeded fixtures, and returns a typed `StepResult`.
- **PlanWorkflow** — Workflow with a plan → execute → evaluate → decide loop, a replan branch, and terminal exit states.
- **PlanEntity** — EventSourcedEntity holding the plan lifecycle, the plan ledger, the step log, and the final outcome.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **PlanView** — projection used by the UI.
- **PlanEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the goal prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Plan` record fields (e.g., add `confidenceScore` per step).
- `prompts/planner.md` — narrow the planner to a single problem domain (e.g., data-pipeline-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-step quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a goal → planner produces a plan, executor runs each step, task completes within ~3 minutes.
2. Inject a step that would violate the executor policy → guardrail blocks the step; planner records the block and replans.
3. Trigger the operator halt while a plan is executing → in-flight step finishes; no further steps are dispatched; plan moves to `HALTED`.
4. Replan budget exhaustion — after two consecutive replan signals the plan moves to `FAILED` with a clear reason.

## License

Apache 2.0.
