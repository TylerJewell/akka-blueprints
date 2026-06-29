# Akka Sample: Self-Discover Workflow

A `PlannerAgent` selects and composes relevant reasoning modules from a fixed module library into a task-specific reasoning plan, evaluates that plan before execution begins, then executes each module step in sequence and synthesises a final answer. Demonstrates the **planner-executor** coordination pattern where planning and execution are explicitly separated and the plan itself is a governed artefact.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Module execution is simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.general.self-discover-planner  ~/my-projects/self-discover-planner
cd ~/my-projects/self-discover-planner
```

(Optional) Edit `SPEC.md` to change the module library, the model provider, or the plan-quality threshold.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that selects a subset of reasoning modules relevant to the task, arranges them in a coherent order, and emits a `ReasoningPlan`. This plan passes through an eval-event before execution begins.
- **ExecutorAgent** — AutonomousAgent that executes one reasoning module at a time and returns a `ModuleResult` for each step.
- **SynthesiserAgent** — AutonomousAgent that combines all module results into a final `TaskAnswer`.
- **PlanEvaluatorAgent** — AutonomousAgent that scores the `ReasoningPlan` against a quality rubric before any execution step runs. Blocking: a plan below threshold is rejected and returned to the `PlannerAgent` for revision.
- **SolveWorkflow** — Workflow driving: compose-plan → evaluate-plan → [reject loop] → execute-modules (one step per module) → synthesise → complete.
- **SolveEntity** — EventSourcedEntity holding the full solve lifecycle, the reasoning plan, per-module results, and the final answer.
- **ModuleRegistry** — EventSourcedEntity holding the canonical list of available reasoning modules (seeded at startup from `sample-data/modules.jsonl`).
- **SolveView** — projection used by the UI.
- **RequestSimulator** — TimedAction that drips a sample task every 90 seconds so the App UI is not empty when first loaded.
- **StuckSolveMonitor** — TimedAction that marks any solve stuck in `EXECUTING` past 5 minutes as `STUCK`.
- **SolveEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task prompts the simulator drips, or adjust the solve timeout.
- `SPEC.md §5` — add fields to `ReasoningPlan` (e.g., a `confidenceScore`).
- `prompts/planner.md` — restrict the module selection to a narrower domain.
- `eval-matrix.yaml` — raise or lower the plan-quality threshold.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → planner selects modules, plan passes evaluation, all modules execute, synthesiser produces an answer within ~3 minutes.
2. Submit a task whose best plan scores below threshold → the evaluator rejects it, the planner revises, and execution only starts after a passing plan is produced.
3. Submit a task and observe that the plan is emitted as a first-class `SolvePlanned` event; the eval score and verdict are recorded in the solve's event log before any module step runs.
4. A module result containing a secret-shaped string is scrubbed before it reaches the synthesiser's input.

## License

Apache 2.0.
