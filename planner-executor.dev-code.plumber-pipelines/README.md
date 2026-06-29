# Akka Sample: Plumber Data Engineering Assistant

A PipelinePlanner agent designs data pipelines — Apache Spark, Beam, or dBT — by drafting a pipeline ledger (sources, transforms, sinks, validation plan, current stage). A suite of specialist executor agents — SparkAgent, BeamAgent, DbtAgent, SchemaAgent — carries out each stage. A progress ledger records every execution outcome. The workflow replans on stage failures and enforces a test gate before the pipeline definition is finalised.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Spark, Beam, and dBT execution surfaces are simulated inside the same Akka service using seeded pipeline fixtures.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.plumber-pipelines  ~/my-projects/plumber-pipelines
cd ~/my-projects/plumber-pipelines
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PipelinePlannerAgent** — AutonomousAgent that maintains a pipeline ledger (sources, transforms, sinks, validation plan, current stage) and a progress ledger (per-stage attempts, verdicts, errors). Decides what runs next. Replans on two consecutive stage failures.
- **SparkAgent** — AutonomousAgent that generates Spark job definitions from seeded fixtures.
- **BeamAgent** — AutonomousAgent that generates Beam pipeline definitions from seeded fixtures.
- **DbtAgent** — AutonomousAgent that generates dBT model SQL and YAML from seeded fixtures.
- **SchemaAgent** — AutonomousAgent that validates source and sink schemas against a fixture schema registry.
- **PipelineWorkflow** — Workflow with a plan → dispatch → test-gate → execute → record → decide loop, replan branch, and terminal exit states.
- **PipelineEntity** — EventSourcedEntity holding the pipeline lifecycle, both ledgers, and the final definition.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **PipelineView** — projection used by the UI.
- **PipelineEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the pipeline prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Pipeline` record fields (e.g., add `slaMinutes`).
- `prompts/pipeline-planner.md` — narrow the planner to one engine (e.g., dBT-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-stage output quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a pipeline request → planner designs it, dispatches to specialists, reaches `FINALISED` within ~3 minutes.
2. Inject a stage that would fail the test gate → gate blocks the stage; planner records the failure and replans.
3. Click **Halt new dispatches** while a pipeline is `EXECUTING` → in-flight stage completes; pipeline moves to `HALTED`.
4. A stage result containing a secret-shaped string is scrubbed before it reaches the planner's next-step prompt.

## License

Apache 2.0.
