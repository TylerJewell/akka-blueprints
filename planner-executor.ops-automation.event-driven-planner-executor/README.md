# Akka Sample: Workflow Integration

An Orchestrator decomposes an operations automation request into a queue of typed steps, dispatches each step to a specialist executor (HTTP, queue publish, database query, script runner), tracks durable execution state on a job ledger and a step ledger, and retries or replans when a step fails. Demonstrates the **planner-executor** coordination pattern wired to event-driven workflow execution with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The four executor surfaces — HTTP calls, queue publish, database query, script execution — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.ops-automation.event-driven-planner-executor  ~/my-projects/event-driven-planner-executor
cd ~/my-projects/event-driven-planner-executor
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any executor's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OrchestratorAgent** — AutonomousAgent that maintains a job ledger (facts, plan, current dispatch) and a step ledger (per-step attempts, verdicts, results). Decides which executor runs each step. Replans on consecutive failures.
- **HttpCallerAgent** — AutonomousAgent that executes HTTP call subtasks from seeded fixtures.
- **QueuePublisherAgent** — AutonomousAgent that simulates publishing events to named queues from seeded fixtures.
- **DbQueryAgent** — AutonomousAgent that returns query results from seeded fixture data.
- **ScriptRunnerAgent** — AutonomousAgent that returns simulated script output for an allow-listed script set.
- **JobWorkflow** — Workflow with a plan → dispatch → execute → sanitize → record → decide loop, retry branch, replan branch, and terminal exit states.
- **JobEntity** — EventSourcedEntity holding the job lifecycle, both ledgers, and the final report.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **JobView** — projection used by the UI.
- **JobEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the job prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Job` record fields (e.g., add `priorityScore`).
- `prompts/orchestrator.md` — narrow the orchestrator to a specific automation domain (e.g., deployment-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-step quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a job → orchestrator plans, dispatches to executors, completes within ~3 minutes.
2. Inject a step policy violation → guardrail blocks the offending dispatch; orchestrator records the block on the step ledger and replans.
3. Trigger the operator halt → no new steps dispatch; in-flight steps finish; job moves to `HALTED`.
4. A step result containing a credential-shaped string is scrubbed before it reaches the orchestrator's next-step prompt.

## License

Apache 2.0.
