# Akka Sample: ML Engineering Pipeline

A PlannerAgent breaks an ML engineering request into a sequenced pipeline, dispatches each stage to one of four specialist agents — DataProfiler, FeatureEngineer, ModelTrainer, ModelEvaluator — tracks progress on a pipeline ledger and an evaluation ledger, and replans when a stage fails or quality gates are not met. Demonstrates the **planner-executor** coordination pattern with embedded governance targeting model eval gating and drift/fairness monitoring.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The four specialist surfaces — data profiling, feature engineering, model training, model evaluation — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.mle-pipeline  ~/my-projects/mle-pipeline
cd ~/my-projects/mle-pipeline
```

(Optional) Edit `SPEC.md` to change the pipeline prompts the simulator drips, or any agent's behavior.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that maintains a pipeline ledger (facts, gaps, stage plan, active dispatch) and an evaluation ledger (per-stage attempts, quality verdicts, gate outcomes, blockers). Decides which specialist runs next. Replans on two consecutive gate failures.
- **DataProfilerAgent** — AutonomousAgent that returns dataset statistics and schema validation results from seeded fixtures.
- **FeatureEngineerAgent** — AutonomousAgent that proposes feature transformations and encoding strategies.
- **ModelTrainerAgent** — AutonomousAgent that returns training metadata (hyperparameters, convergence curves) from seeded fixtures.
- **ModelEvaluatorAgent** — AutonomousAgent that scores a trained model against held-out eval sets and returns metric bundles.
- **PipelineWorkflow** — Workflow with a plan → dispatch → execute → evaluate → gate → record → decide loop, replan branch, and terminal exit states.
- **PipelineRunEntity** — EventSourcedEntity holding the pipeline run lifecycle, both ledgers, and the final model report.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag. Single instance keyed by `"global"`.
- **RunQueue** — EventSourcedEntity serving as the audit log of submitted pipeline runs.
- **PipelineRunView** — projection used by the UI.
- **RunRequestConsumer** — Consumer that starts a `PipelineWorkflow` per submission.
- **RunSimulator** — TimedAction dripping sample pipeline requests every 90 seconds.
- **StaleRunMonitor** — TimedAction marking any run stuck in `EXECUTING` past 10 minutes as `STUCK`.
- **PipelineEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the pipeline prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `PipelineRun` record fields (e.g., add `targetMetric`).
- `prompts/planner.md` — narrow the planner to a single model family or evaluation protocol.
- `eval-matrix.yaml` — add a `sanitizer` control if you want sensitive column names scrubbed from evaluation reports.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an ML pipeline request → planner stages the work, dispatches to specialists, run completes within ~5 minutes with a model report.
2. Inject an evaluation gate failure → the gate blocks promotion of the model; planner records the failure and either replans or ends in `GATE_FAILED`.
3. Trigger the operator halt → no new stage dispatches; in-flight stages finish; run moves to `HALTED`.
4. A drift/fairness watch fires on a periodic evaluation run → a `DriftAlertRaised` event is recorded; the UI surfaces the alert on the run card.

## License

Apache 2.0.
