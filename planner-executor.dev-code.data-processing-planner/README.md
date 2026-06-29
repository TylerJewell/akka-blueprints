# Akka Sample: Amazon DataProcessing Agent

An agent that plans and executes data-processing pipelines on AWS-style infrastructure. A PlannerAgent decomposes a pipeline request into an ordered job plan, dispatches each job step to a JobExecutorAgent, tracks outcomes on a job ledger and run ledger, enforces a cost guardrail before each submission, and halts on operator command. Demonstrates the **planner-executor** coordination pattern with embedded governance for infrastructure-consuming workloads.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Job submissions to Glue, EMR, and Spark clusters are simulated inside the same Akka service using seeded fixtures and canned execution results.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.data-processing-planner  ~/my-projects/data-processing-planner
cd ~/my-projects/data-processing-planner
```

(Optional) Edit `SPEC.md` to change the pipeline prompts the simulator drips, adjust cost thresholds, or swap in a real AWS client.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that maintains a job ledger (known inputs, missing parameters, ordered job plan, current dispatch) and a run ledger (per-step attempts, verdicts, observed output). Decides whether to continue, revise the plan, complete, or fail. Replans on two consecutive step failures.
- **JobExecutorAgent** — AutonomousAgent that runs a single pipeline job step (Glue crawler, EMR step, Spark submit, or S3 copy) against seeded fixtures and returns a typed `JobStepResult`.
- **PipelineWorkflow** — Workflow with a plan → dispatch-guarded → execute → record → decide loop, replan branch, and terminal exit states.
- **PipelineEntity** — EventSourcedEntity holding the pipeline run's lifecycle, both ledgers, and the final output manifest.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag, keyed by literal `"global"`.
- **PipelineQueue** — EventSourcedEntity acting as the audit log of submitted pipeline requests.
- **PipelineView** — View projection consumed by the UI.
- **PipelineRequestConsumer** — Consumer that starts a `PipelineWorkflow` per queue entry.
- **PipelineSimulator** — TimedAction that drips a sample pipeline request every 90 s.
- **StuckPipelineMonitor** — TimedAction that marks any run stuck in `RUNNING` past 8 minutes as `STUCK`.
- **PipelineEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the pipeline prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `PipelineRun` record fields (e.g., add `estimatedCostUsd`).
- `prompts/planner.md` — narrow the planner to a single engine type (e.g., Glue-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-step quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a pipeline request → planner decomposes it, executor runs each step, run ends in `COMPLETED` within ~4 minutes.
2. Inject a job step that would exceed the cost guardrail → guardrail blocks the dispatch; planner either finds a cheaper alternative or the run ends in `FAILED`.
3. Trigger the operator halt → no new job steps dispatch; the in-flight step finishes; run moves to `HALTED`.
4. Submit a pipeline whose fixture output contains a credential-shaped string → sanitizer scrubs it before it reaches the planner's next prompt.

## License

Apache 2.0.
