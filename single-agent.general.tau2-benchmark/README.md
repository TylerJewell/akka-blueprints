# Akka Sample: Tau2 Benchmark Agent

A single evaluation agent runs agent-benchmark tasks from the tau2 suite, records each task's result, and scores overall performance across task categories. Task definitions are submitted one at a time; the agent executes each task's steps, returns a structured `TaskResult`, and a rule-based scorer produces a `BenchmarkScore` immediately after each result lands.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: an on-decision periodic evaluator that collects per-task results, computes aggregate performance metrics, and flags capability gaps — without a second LLM call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the benchmark task corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.general.tau2-benchmark  ~/my-projects/tau2-benchmark
cd ~/my-projects/tau2-benchmark
```

(Optional) Edit `SPEC.md` to point at a different task corpus or swap in additional task categories beyond the seeded web-navigation, tool-use, and multi-step-reasoning sets.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BenchmarkAgent** — an AutonomousAgent that accepts a task definition as an attachment and returns a typed `TaskResult`.
- **BenchmarkRunWorkflow** — orchestrates load-task → execute → score per submitted run.
- **BenchmarkRunEntity** — an EventSourcedEntity holding the per-run lifecycle.
- **TaskScorer** — a deterministic rule-based scorer that runs inside the workflow after each result; no LLM call.
- **BenchmarkView + BenchmarkEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded task corpus for your own (the JSONL file under `src/main/resources/sample-events/tasks.jsonl` after generation).
- `SPEC.md §5` — extend `TaskResult` with additional fields (e.g., `stepTrace`, `toolCallCount`, `latencyMs`).
- `prompts/benchmark-agent.md` — narrow the agent's behavior for a specific task type (e.g., constrain it to web-navigation only).
- `eval-matrix.yaml` — adjust the periodic evaluation thresholds (pass-rate floor, per-category caps) to match your performance bar.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task definition → the agent executes it → the result appears in the UI with a score.
2. A run that produces a `TaskResult` with all empty `stepOutputs` receives a performance score of 1 and a flagged card.
3. The aggregate performance report updates in real time as successive task runs land.
4. A task whose execution exceeds the step timeout transitions the run to `FAILED` and preserves partial state for inspection.

## License

Apache 2.0.
