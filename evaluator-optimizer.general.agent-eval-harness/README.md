# Akka Sample: Agent Eval Harness

An evaluator agent runs a suite of test cases against a target agent; a judge agent scores each response; the harness aggregates pass rates, records every verdict as a typed eval event, and gates the workflow on a configurable accuracy threshold. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance via periodic accuracy measurement and CI-grade quality gates.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The harness ships with a built-in sample test suite and runs out of the box.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.agent-eval-harness  ~/my-projects/agent-eval-harness
cd ~/my-projects/agent-eval-harness
```

(Optional) Edit `SPEC.md` to change the sample test suite, the accuracy threshold, the per-run case limit, or the regression baseline.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **EvaluatorAgent** — AutonomousAgent that invokes the target agent on each test case from the active suite, collecting the raw response.
- **JudgeAgent** — AutonomousAgent that scores each response against the ground-truth answer and rubric, returning `PASS` or `FAIL` with a typed `JudgingNotes` payload.
- **EvalRunWorkflow** — Workflow that drives the evaluate → judge loop over all cases in the suite, aggregates the pass rate, checks it against the accuracy threshold, and transitions the run to `PASSED` or `FAILED`.
- **EvalRunEntity** — EventSourcedEntity that holds the run lifecycle, every case result, and the final accuracy score.
- **SuiteRegistry** — EventSourcedEntity that stores test suite definitions (name, cases, expected answers, rubric).
- **EvalRunsView** — read-side projection that the UI lists and streams via SSE.
- **SuiteEventConsumer** — Consumer that starts a new workflow whenever a suite is published or triggered.
- **RunScheduler** — TimedAction that triggers a periodic accuracy run every 5 minutes on the default suite so the App UI is never empty.
- **AccuracySampler** — TimedAction that records a periodic `AccuracySnapshot` event for any completed run since the last tick.
- **EvalEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the test cases the scheduler submits, or disable the scheduler entirely.
- `SPEC.md §5` — adjust the `EvalRun` record fields (e.g., raise `passingThreshold`, change `maxCasesPerRun`).
- `prompts/evaluator.md` — adjust how the evaluator calls the target agent (prompt, few-shot examples, output format).
- `prompts/judge.md` — change the rubric (e.g., add a factual-citation dimension).
- `eval-matrix.yaml` — tighten the CI gate threshold or change the accuracy-snapshot period.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Trigger a run → all cases are evaluated, the run transitions `PENDING` → `RUNNING` → `PASSED` or `FAILED` with a final accuracy score.
2. Force accuracy below the threshold → run lands in `FAILED` with a structured failure reason and all case results preserved.
3. The CI gate control blocks a run that fails to reach the threshold, recording a `GateFailed` event the operator can inspect.
4. Each completed run emits an `AccuracySnapshot` event that surfaces in the App UI's run timeline.

## License

Apache 2.0.
