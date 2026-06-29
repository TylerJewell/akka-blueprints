# Akka Sample: SK Process

A process-coordinator agent breaks an incoming automation job into ordered steps, dispatches each step to a pool of specialized execution agents, monitors progress via a runtime dashboard, and supports safe pause and graceful stop at any point. Demonstrates the **composite-multi-team** coordination pattern applied to long-running distributed operational workflows.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. The coordinator, executor pool, monitor, and job intake are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./composite-multi-team.ops-automation.durable-process-orchestration  ~/my-projects/sk-process
cd ~/my-projects/sk-process
```

(Optional) Edit `SPEC.md` to change the process templates the simulator drips, the executor roster, the monitoring thresholds, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ProcessCoordinator** — AutonomousAgent that plans a `StepPlan` from a `JobRequest` and, at completion, assembles the `JobSummary` under an output guardrail.
- **StepPlanner + StepExecutor** — the planning desk: the planner decomposes the job into ordered steps; executor instances each claim one step off the shared step board and run it. Executors self-organising around a shared step list is the team capability.
- **QualityChecker** — the validation desk: several checker instances each evaluate the output of one completed step on a distinct criterion (format, completeness, policy), and a deterministic `QualityRule` turns their verdicts into an overall pass-or-retry outcome. A scored panel feeding an aggregation rule is the moderation capability.
- **ProcessOrchestrationWorkflow** — the coordinator pipeline: plan → execute → validate → finalise, with a retry loop when validation requests a redo.
- **ExecutorWorkflow** — per-executor durable loop: poll the step board → claim a step → run the executor agent → mark step done.
- **JobEntity + StepEntity** — the shared job workspace and the per-step board entry whose atomic claim prevents two executors grabbing the same step.
- **JobBoardView, StepBoardView** — the read models the UI and the workflow poll.
- **JobRequestConsumer** — listens to `JobQueue` events; starts a `ProcessOrchestrationWorkflow` per submission.
- **StepEvalConsumer** — fires a non-blocking quality signal on every step completion.
- **JobQueueEndpoint + AppEndpoint** — REST + SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the process templates the simulator drips, or remove the simulator.
- `SPEC.md §11` — change the executor roster (`executor-1`, `executor-2`, `executor-3`) or the validation criteria (`format`, `completeness`, `policy`).
- `prompts/step-executor.md`, `prompts/quality-checker.md` — narrow each role to a specific operation domain.
- `eval-matrix.yaml` — adjust the runtime-monitoring thresholds or the graceful-degradation criteria.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a job → the coordinator plans steps → executors claim and run steps off a shared board → quality checkers validate outputs → a passing verdict assembles and finalises the job result. The board updates live via SSE.
2. Two executors poll at the same instant → each step ends up claimed by exactly one executor (no double-claim).
3. A quality checker returns a retry verdict → the affected step resets to open and an executor rewrites it; the pipeline terminates after one bounded retry round.
4. An executor attempts a write outside its assigned step → the before-tool-call guardrail blocks it and the step is recorded with the guardrail reason.
5. A running job receives a pause signal → the workflow enters `PAUSED` state and the operator dashboard shows the pause; resume returns it to the same step without data loss.
6. An operator posts a post-completion audit review → it is recorded against the finished job without changing its completed state.

## License

Apache 2.0.
