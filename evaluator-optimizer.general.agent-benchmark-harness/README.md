# Akka Sample: Agent Benchmark Harness

A task-runner agent submits evaluation tasks to a target agent; a scorer agent grades each response against a reference rubric; the harness aggregates the results into a pass/fail report and optionally gates a deployment pipeline on the outcome. Demonstrates the **evaluator-optimizer** coordination pattern applied to automated quality measurement.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the task set and the reference answers are bundled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.agent-benchmark-harness  ~/my-projects/agent-benchmark-harness
cd ~/my-projects/agent-benchmark-harness
```

(Optional) Edit `SPEC.md` to change the task set, the scoring rubric, the passing threshold, or the per-run attempt ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RunnerAgent** — AutonomousAgent that submits each evaluation task to the target model and collects a raw response.
- **ScorerAgent** — AutonomousAgent that grades a raw response against the reference answer and rubric, returning a `ScoredResult` with a numeric score and pass/fail verdict.
- **BenchmarkRunWorkflow** — Workflow that fans out all tasks in a run, collects individual scores, aggregates the run-level metrics, and transitions the run to `PASSED` or `FAILED` based on the configured threshold.
- **RunEntity** — EventSourcedEntity that holds the benchmark run lifecycle, every task's response and score, and the final aggregate metrics.
- **TaskRegistry** — EventSourcedEntity that stores the canonical task set and reference answers used for each run.
- **RunsView** — read-side projection that the UI lists and streams via SSE.
- **TaskResultConsumer** — Consumer that reacts to task-scored events and updates the run's aggregate counters.
- **RunScheduler** — TimedAction that kicks off a scheduled benchmark run every configurable interval (default 24 h) so the harness can track quality drift over time.
- **AccuracySampler** — TimedAction that records a periodic accuracy snapshot event every 15 minutes (control E1).
- **BenchmarkEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task set loaded from `sample-events/benchmark-tasks.jsonl`, or remove the `RunScheduler` to run benchmarks on demand only.
- `SPEC.md §5` — adjust the `BenchmarkRun` record fields (e.g., raise `passingThreshold`, increase `maxTasksPerRun`).
- `prompts/runner.md` — narrow the runner's instruction format for the target model you are evaluating.
- `prompts/scorer.md` — change the grading rubric (e.g., add semantic-equivalence scoring, partial credit).
- `eval-matrix.yaml` — tighten the CI gate threshold or switch the eval-periodic control to a tighter cadence.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Trigger a run → the run progresses `PENDING` → `RUNNING` → `PASSED` or `FAILED`; every task's response and score are visible in the App UI.
2. Force-fail threshold → a run where every task scores below the passing mark lands in `FAILED` with the aggregate metrics preserved.
3. The CI gate control blocks a deployment action when the most recent run's pass rate is below the configured threshold.
4. Each completed scoring cycle emits an `AccuracySnapshotRecorded` event visible in the per-run timeline.

## License

Apache 2.0.
