# Akka Sample: Trajectory Evaluation

A runner agent executes a task by calling a sequence of tools; an evaluator agent compares that tool-use trajectory against a stored reference path, identifies deviations, and returns a pass/fail verdict with deviation details. The workflow retries the runner up to a configurable ceiling when the trajectory diverges. Demonstrates the **evaluator-optimizer** coordination pattern applied to agent-behavior regression testing.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the runner task inputs and reference trajectories are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.trajectory-eval  ~/my-projects/trajectory-eval
cd ~/my-projects/trajectory-eval
```

(Optional) Edit `SPEC.md` to change the task scenarios the simulator drips, adjust the deviation-threshold, or alter the retry ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RunnerAgent** — AutonomousAgent that executes a task by calling an ordered sequence of tools, producing a recorded trajectory of tool calls and their outputs.
- **EvaluatorAgent** — AutonomousAgent that compares a recorded trajectory against the reference path, identifies each deviation by step index, and returns a `TrajectoryVerdict` of `PASS` or `FAIL` with typed `DeviationReport` payload.
- **TrajectoryWorkflow** — Workflow that runs the execute → evaluate → retry loop up to a configurable ceiling, transitions the evaluation to `PASSED` on a `PASS` verdict or to `FAILED_FINAL` when the ceiling is exhausted.
- **EvaluationEntity** — EventSourcedEntity that holds the evaluation lifecycle, every recorded trajectory, every deviation report, and the final outcome.
- **ReferenceStore** — EventSourcedEntity that holds the set of reference paths keyed by task-scenario id; supports add, update, and retrieval.
- **EvaluationsView** — read-side projection that the UI lists and streams via SSE.
- **ScenarioConsumer** — Consumer that starts a workflow per inbound scenario submission.
- **ScenarioSimulator** — TimedAction that drips a sample task scenario every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records a periodic eval event each cycle (control E1).
- **TrajectoryEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task scenarios the simulator drips, or disable the simulator entirely.
- `SPEC.md §5` — adjust the `Evaluation` record fields (e.g., raise `maxAttempts`, tighten the deviation threshold).
- `prompts/runner.md` — change which tool-call format the runner produces, or swap the task domain.
- `prompts/evaluator.md` — change the comparison rubric (e.g., require strict ordering, or allow reordered steps as partial matches).
- `eval-matrix.yaml` — adjust enforcement on the periodic monitor (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a scenario whose reference path the runner matches on the first attempt — evaluation reaches `PASSED`.
2. Submit a scenario configured to produce a trajectory mismatch every run — evaluation exhausts `maxAttempts` and lands in `FAILED_FINAL`, with every trajectory and deviation report preserved.
3. The output guardrail blocks a trajectory that exceeds the maximum allowed step count before the evaluator runs, triggering a re-run.
4. Each completed evaluation cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
