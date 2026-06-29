# Akka Sample: Plan-and-Execute Agent

A Planner LLM decomposes a goal into a numbered plan; an Executor agent carries out each step with a scoped tool set; after every observation a Replanner decides whether to continue, revise the plan, or conclude. Demonstrates the **planner-executor** coordination pattern with a before-tool-call guardrail and per-replanning quality evaluation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Tool calls are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.general.plan-execute-replan  ~/my-projects/plan-execute-replan
cd ~/my-projects/plan-execute-replan
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that converts a user goal into a typed `ExecutionPlan` (numbered steps, each with a tool assignment and expected output).
- **ExecutorAgent** — AutonomousAgent that carries out a single plan step by invoking one of the registered tools (search, read, calculate, summarise) and returning a typed `StepResult`.
- **ReplannerAgent** — AutonomousAgent that reads the goal, the current plan, and the accumulated observations, then returns a `ReplanDecision` — continue, revise plan, or conclude.
- **ExecutionWorkflow** — Workflow that drives the plan → execute → observe → replan loop, replan branch, and terminal exit states.
- **GoalEntity** — EventSourcedEntity holding the goal's lifecycle, current plan, accumulated observations, and the final conclusion.
- **SystemControlEntity** — EventSourcedEntity holding the operator pause flag. Single instance keyed by literal `"global"`.
- **GoalView** — projection consumed by the UI.
- **GoalRequestConsumer** — Consumer that starts one `ExecutionWorkflow` per submitted goal.
- **GoalQueue** — EventSourcedEntity audit log of submitted goals.
- **RequestSimulator** — TimedAction that drips a sample goal every 90 s.
- **StuckGoalMonitor** — TimedAction that marks goals stuck in `EXECUTING` past 5 minutes as `STUCK`.
- **GoalEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the goal prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Goal` record fields (e.g., add `confidenceThreshold`).
- `prompts/planner.md` — narrow the planner to a specific domain (e.g., research-only).
- `eval-matrix.yaml` — add a `guardrail` control for the replanner's revised plan if you want pre-execution vetting of replanned steps as well.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a goal → planner produces a plan, executor runs each step, replanner concludes within ~3 minutes.
2. Inject a tool call outside the allow-list → guardrail blocks the step; replanner records the block and revises; task either completes via an alternate path or fails after the replan budget is exhausted.
3. Trigger the operator pause → no new steps execute; the in-flight step finishes; goal moves to `PAUSED`.
4. The replanner evaluates a poor-quality revised plan and emits a quality score below threshold; the eval event is recorded; the plan is flagged before the next step runs.

## License

Apache 2.0.
