# Akka Sample: Graph Pattern

A single `GraphAgent` walks a task request through an explicit DAG of processing nodes — **PARSE → PLAN → EXECUTE nodes in topological order → MERGE** — where each node in the execution layer runs once its declared predecessors have completed. Each node has its own typed input, typed output, and a bounded tool set. The user submits a task description and receives a typed `TaskResult`.

Demonstrates the **sequential-pipeline** coordination pattern applied to a DAG-structured workload: a topologically-ordered sequence of agent node calls wired together by explicit data-dependency edges, with two governance mechanisms: a `before-tool-call` guardrail that rejects tool calls when the calling node's declared predecessors have not yet recorded their outputs, and an `on-decision-eval` evaluator that scores every emitted `TaskResult` for node coverage and output traceability.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every parse, plan, execute, and merge tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.general.graph-pattern  ~/my-projects/graph-pattern
cd ~/my-projects/graph-pattern
```

(Optional) Edit `SPEC.md` to change the seeded task set under `src/main/resources/sample-events/tasks.jsonl`, a different model provider, or a richer set of execution tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GraphAgent** — one AutonomousAgent declaring four Task constants (`PARSE_REQUEST`, `PLAN_GRAPH`, `EXECUTE_NODES`, `MERGE_OUTPUTS`); the workflow runs them in order, feeding each typed result forward as the next task's instruction context.
- **GraphExecutionWorkflow** — runs `parseStep → planStep → executeStep → mergeStep → evalStep`. Each step calls `runSingleTask`, writes the typed result back onto `GraphRunEntity`, then advances.
- **GraphRunEntity** — an EventSourcedEntity holding the per-run lifecycle (`RequestParsed`, `GraphPlanned`, `NodeExecuted`, `OutputsMerged`, `EvaluationScored`).
- **ParseTools / PlanTools / ExecuteTools / MergeTools** — four function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase.
- **DependencyGuardrail** — the runtime check that backs the DAG dependency contract. A tool call referencing a node whose predecessor nodes have not yet recorded their outputs is rejected before the tool runs.
- **CoverageScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `OutputsMerged` and emits a 1–5 score.
- **GraphRunView + GraphEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded task set in `src/main/resources/sample-events/tasks.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/graph-agent.md` — narrow the agent's role (e.g., constrain it to code-review pipelines, to document-generation DAGs, to multi-step data-transformation workflows) by tightening the system prompt and renaming the typed records.
- `SPEC.md §5` — extend the typed outputs (`ParsedRequest`, `GraphPlan`, `NodeOutput`, `MergedResult`) with domain-specific fields. The dependency-gating guardrail does not need editing — it checks recorded-predecessor preconditions, not field types.
- `eval-matrix.yaml` — wire a real output-traceability evaluator by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task description → `PARSE` runs → `PLAN` runs → `EXECUTE` runs nodes in topological order → `MERGE` runs → a typed `TaskResult` lands in the UI within ~60 s. Every node transition is visible in real time.
2. An execute-phase tool is called before all predecessor nodes have recorded outputs (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries → the run completes correctly.
3. Every `TaskResult` emitted has an on-decision eval score visible on the same UI card; results that reference a node output absent from the recorded `GraphPlan` receive a score ≤ 2 and are flagged.
4. Each task receives only its typed inputs; the PARSE task does not see the merge instructions, and the MERGE task does not see the raw parse output — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
