# Akka Sample: BigQuery Pipeline Builder

A Planner decomposes a natural-language pipeline request into ordered build steps, dispatches each step to a specialist executor (Schema Analyst, SQL Composer, Dataform Modeler, Validator), tracks progress on a pipeline ledger plus a step ledger, and replans when a build step fails. Demonstrates the **planner-executor** coordination pattern applied to data-engineering pipeline construction with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** BigQuery and Dataform surfaces are simulated inside the same Akka service using seeded fixture datasets and SQL fixtures; no live GCP project is needed.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.bq-pipeline-builder  ~/my-projects/bq-pipeline-builder
cd ~/my-projects/bq-pipeline-builder
```

(Optional) Edit `SPEC.md` to change the pipeline domain, model provider, or any executor's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that maintains a pipeline ledger (dataset facts, schema gaps, build plan, dispatch decision) and a step ledger (per-step attempts, verdicts, blockers). Decides which executor runs next. Replans on three consecutive build-step failures.
- **SchemaAnalystAgent** — AutonomousAgent that reads seeded BigQuery schema fixtures and reports table/field layouts.
- **SqlComposerAgent** — AutonomousAgent that drafts or refines SQL queries and transformations.
- **DataformModelerAgent** — AutonomousAgent that produces Dataform SQLX model definitions and `includes/` JS macros.
- **ValidatorAgent** — AutonomousAgent that runs deterministic lint and dry-run checks against fixture schema, returning a `ValidationReport`.
- **PipelineWorkflow** — Workflow with a plan → dispatch → execute → validate → record → decide loop, replan branch, and terminal exit states.
- **PipelineEntity** — EventSourcedEntity holding the pipeline lifecycle, both ledgers, and the final build manifest.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **PipelineQueue** — EventSourcedEntity audit log of submitted pipeline requests.
- **PipelineView** — projection used by the UI.
- **PipelineRequestConsumer** — Consumer that starts a `PipelineWorkflow` per submission.
- **PipelineSimulator** — TimedAction that drips a sample request every 90 seconds.
- **StalePipelineMonitor** — TimedAction that marks stuck-in-progress pipelines as `STUCK`.
- **PipelineEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the pipeline prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Pipeline` record fields (e.g., add a `targetDataset` field).
- `prompts/planner.md` — narrow the planner to a specific domain (e.g., marketing-analytics pipelines only).
- `eval-matrix.yaml` — add a `sanitizer` control if you want credential scrubbing on SQL outputs.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit "Build a pipeline that loads raw GA4 events into a clean sessions table." Pipeline progresses `PLANNING → BUILDING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a pipeline ledger with a non-empty build plan, a step ledger with 3–8 entries, and a non-empty `BuildManifest`.
2. Submit a pipeline whose plan would execute destructive DDL (`DROP TABLE`) — the DDL guardrail blocks the step; the planner records the block and replans.
3. Submit a pipeline and click **Halt new dispatches** while it is `BUILDING`. The in-flight step finishes; no further dispatches occur; the pipeline ends in `HALTED`.
4. Force validation to fail three consecutive times on the same step — the planner exhausts its failure budget and the pipeline ends in `FAILED` with a clear `failureReason`.

## License

Apache 2.0.
