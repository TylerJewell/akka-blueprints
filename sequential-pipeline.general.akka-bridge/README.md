# Akka Sample: Functional API Plugin

Each node in a graph-execution engine produces an LLM response. This sample wraps each node as an Akka workflow activity — giving every node durable execution, automatic retries, and crash-recovery — while an `after-llm-response` guardrail checks every node's output before it is forwarded to the next node in the graph.

Demonstrates the **sequential-pipeline** coordination pattern with one governance mechanism: an `after-llm-response` guardrail that validates each node's output against a content policy before the workflow advances to the downstream node.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every node execution tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.general.akka-bridge  ~/my-projects/akka-bridge
cd ~/my-projects/akka-bridge
```

(Optional) Edit `SPEC.md` to point at a different graph definition, a different model provider, or a richer output-validation policy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GraphAgent** — one AutonomousAgent declaring three Task constants (`PLAN_GRAPH`, `EXECUTE_NODE`, `FINALIZE_OUTPUT`); the workflow runs them in order per graph execution, feeding each node's typed output forward as the next node's instruction context.
- **GraphExecutionWorkflow** — runs `planStep → executeStep → finalizeStep → guardStep`. Each step calls `runSingleTask` and writes the typed result back onto `GraphRunEntity` before the next step starts.
- **GraphRunEntity** — an EventSourcedEntity holding the per-run lifecycle (`GraphPlanned`, `NodeExecuted`, `OutputFinalized`, `GuardrailChecked`).
- **PlanTools / ExecuteTools / FinalizeTools** — three function-tool classes registered on the agent, one per phase. The `after-llm-response` guardrail intercepts each node's LLM output before it flows downstream.
- **OutputGuardrail** — the runtime check that sits after each node's LLM response. A node output that violates the content policy is blocked and the workflow records the violation event.
- **PolicyChecker** — deterministic, rule-based after-llm-response evaluator that runs immediately after `OutputFinalized` and emits a 1–5 quality score.
- **GraphRunView + GraphEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded graph definitions under `src/main/resources/sample-events/graphs.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/graph-agent.md` — narrow the agent's role (e.g., constrain it to code-generation graphs, data-transformation pipelines, or document-processing chains) by tightening the system prompt and renaming the typed records (`NodeOutput`, `ExecutionPlan`, `FinalResult`).
- `SPEC.md §5` — extend the typed outputs (`ExecutionPlan`, `NodeOutput`, `FinalResult`) with domain-specific fields. The after-llm-response guardrail does not need editing — it checks recorded output against the policy, not field shapes.
- `eval-matrix.yaml` — wire a real content-validation guardrail (replace the policy-check stub with a vector-similarity check against a prohibited-content registry) by editing the `H1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a graph definition → `PLAN` runs → `EXECUTE` runs → `FINALIZE` runs → a typed `FinalResult` lands in the UI within ~60 s. Every transition is visible in real time.
2. A node's LLM output contains a policy-violating pattern (forced via the mock LLM) → the `after-llm-response` guardrail blocks the output → the workflow records the violation event → the agent retries within its iteration budget → the pipeline completes correctly.
3. Every `FinalResult` emitted has an on-decision eval score visible on the same UI card; results with no node outputs or empty content receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the PLAN task does not see the finalized output, and the FINALIZE task does not see raw plan instructions — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
