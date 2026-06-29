# Akka Sample: Workflows Process Orchestration

A single `PipelineOrchestrationAgent` drives multi-step operations through three stage phases — **VALIDATE → EXECUTE → NOTIFY** — wired together by explicit stage dependencies. Each stage has its own typed input, typed output, and a focused set of stage-specific tools. An operator submits a workflow run request and receives a structured `RunResult`.

Demonstrates the **sequential-pipeline** coordination pattern with two governance mechanisms: a `before-tool-call` guardrail that blocks tools when their stage's preconditions are not yet met, and an `on-completion-eval` evaluator that scores every emitted run result for correctness and step coverage.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every validate / execute / notify tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.ops-automation.style-workflow  ~/my-projects/workflow-orchestration
cd ~/my-projects/workflow-orchestration
```

(Optional) Edit `SPEC.md` to point at a different workflow definition catalog, a different model provider, or a richer set of execution tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PipelineOrchestrationAgent** — one AutonomousAgent declaring three Task constants (`VALIDATE_STEPS`, `EXECUTE_STEPS`, `NOTIFY_COMPLETION`); the workflow runs them in order, feeding each output forward as the next stage's instruction context.
- **WorkflowRunPipeline** — runs `validateStage → executeStage → notifyStage → evalStage`. Each stage calls `runSingleTask` and writes the typed result back onto `WorkflowRunEntity` before the next stage starts.
- **WorkflowRunEntity** — an EventSourcedEntity holding the per-run lifecycle (`StepsValidated`, `StepsExecuted`, `NotificationSent`, `RunEvaluated`).
- **ValidateTools / ExecuteTools / NotifyTools** — three function-tool classes registered on the agent, one per stage. The `before-tool-call` guardrail enforces that each tool is only callable in its own stage.
- **StageGuardrail** — the runtime check that backs the dependency contract. A tool call referencing a stage whose precondition has not been recorded on the entity is rejected before the tool runs.
- **StepCoverageScorer** — deterministic, rule-based on-completion evaluator that runs immediately after `NotificationSent` and emits a 1–5 score.
- **WorkflowRunView + WorkflowRunEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded workflow definition set under `src/main/resources/sample-events/workflows.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/pipeline-orchestration-agent.md` — narrow the agent's role (e.g., constrain it to infrastructure provisioning workflows, to deployment pipelines, to data-migration workflows) by tightening the system prompt and renaming the typed records.
- `SPEC.md §5` — extend the typed outputs (`ValidationReport`, `ExecutionResult`, `RunResult`) with domain-specific fields. The stage-gating guardrail does not need editing — it checks recorded-stage preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real step-coverage evaluator (replace the deterministic stub with a comparison against the declared workflow definition) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. An operator submits a workflow run → `VALIDATE` runs → `EXECUTE` runs → `NOTIFY` runs → a typed `RunResult` lands in the UI within ~60 s. Every transition is visible in real time.
2. A tool from a later stage is called out of order (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries in-stage → the pipeline completes correctly.
3. Every `RunResult` emitted has an on-completion eval score visible on the same UI card; runs whose executed steps do not cover all declared steps receive a score ≤ 2 and are flagged.
4. Each stage receives only its own typed inputs; the VALIDATE stage does not see the notification template, and the NOTIFY stage does not see raw validation checks — the workflow's stage-chaining is the only path information travels between stages.

## License

Apache 2.0.
