# Akka Sample: Sequential Workflow

A single `WorkflowAgent` walks a submitted job request through four task phases — **VALIDATE → ENRICH → EXECUTE → SUMMARIZE** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a set of phase-specific tools. The user submits a job definition and receives a structured `JobResult`.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that blocks tools when their phase's preconditions are not yet met, and an `on-completion-eval` evaluator that scores every emitted job result for output completeness and step traceability.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every validate / enrich / execute / summarize tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.general.sequential-workflow  ~/my-projects/sequential-workflow
cd ~/my-projects/sequential-workflow
```

(Optional) Edit `SPEC.md` to point at a different job type catalog, a different model provider, or a richer set of execution tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WorkflowAgent** — one AutonomousAgent declaring four Task constants (`VALIDATE_JOB`, `ENRICH_JOB`, `EXECUTE_JOB`, `SUMMARIZE_JOB`); the pipeline runs them in order, feeding each output forward as the next task's instruction context.
- **JobPipelineWorkflow** — runs `validateStep → enrichStep → executeStep → summarizeStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `JobEntity` before the next step starts.
- **JobEntity** — an EventSourcedEntity holding the per-job lifecycle (`JobValidated`, `JobEnriched`, `JobExecuted`, `JobSummarized`, `QualityScored`).
- **ValidateTools / EnrichTools / ExecuteTools / SummarizeTools** — four function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase.
- **StepGuardrail** — the runtime check that backs the dependency contract. A tool call referencing a phase whose precondition has not yet been recorded on the entity is rejected before the tool runs.
- **QualityScorer** — deterministic, rule-based on-completion evaluator that runs immediately after `JobSummarized` and emits a 1–5 score.
- **JobView + JobEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded job type catalog under `src/main/resources/sample-events/job-types.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/workflow-agent.md` — narrow the agent's role (e.g., constrain it to data-transformation jobs, report-generation jobs, or approval workflows) by tightening the system prompt and renaming the typed records (`JobSpec`, `EnrichedJob`, `JobOutput`).
- `SPEC.md §5` — extend the typed outputs (`ValidationResult`, `EnrichedJob`, `JobOutput`, `JobSummary`) with domain-specific fields. The phase-gating guardrail does not need editing — it checks recorded-phase preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real output-quality evaluator (replace the deterministic stub with a rule set matched to your job domain) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a job → `VALIDATE` runs → `ENRICH` runs → `EXECUTE` runs → `SUMMARIZE` runs → a typed `JobResult` lands in the UI within ~80 s. Every transition is visible in real time.
2. A tool from a later phase is called out of order (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. Every `JobResult` emitted has an on-completion eval score visible on the same UI card; results with missing output steps or unresolved inputs receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the VALIDATE task does not see execute instructions, and the EXECUTE task does not see raw validation details — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
