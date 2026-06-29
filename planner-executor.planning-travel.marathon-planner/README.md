# Akka Sample: Marathon Training Planner

A TrainingPlannerAgent assesses a runner's goal and baseline fitness, generates a periodized multi-phase training plan, dispatches individual workout builds to a WorkoutBuilderAgent, scores each plan adjustment against the runner's goal signal, and replans when adherence drops. Demonstrates the **planner-executor** coordination pattern with dynamic skill loading governed by a capability registry.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Training data, fitness fixtures, and workout templates are seeded inside the same Akka service.

## Generate the system

```sh
cp -r ./planner-executor.planning-travel.marathon-planner  ~/my-projects/marathon-planner
cd ~/my-projects/marathon-planner
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TrainingPlannerAgent** — AutonomousAgent that assesses a runner's goal, builds a periodized plan ledger, decides workout adjustments, and composes the final plan summary.
- **WorkoutBuilderAgent** — AutonomousAgent that expands plan-phase stubs into full week-by-week workout sessions.
- **FitnessAssessorAgent** — AutonomousAgent that evaluates incoming fitness data and produces a `FitnessBaseline`.
- **PlanWorkflow** — Workflow with an assess → plan → build-workouts → validate → decide loop, replan branch on low adherence, and terminal exit states.
- **RunnerEntity** — EventSourcedEntity holding the runner profile, training plan, and adjustment history.
- **SkillRegistryEntity** — EventSourcedEntity holding the loaded skill capabilities; consulted by the guardrail before every agent dispatch.
- **PlanQueue** — EventSourcedEntity; audit log of plan requests.
- **PlanView** — projection used by the UI.
- **PlanRequestConsumer** — Consumer that subscribes to PlanQueue events and starts a PlanWorkflow per request.
- **PlanSimulator** — TimedAction that drips a sample runner profile every 90 s.
- **StaleWorkoutMonitor** — TimedAction that marks plans in `BUILDING` past 5 minutes as `STALE`.
- **PlanEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the marathon distances or phase structure the simulator uses.
- `SPEC.md §5` — add a `confidenceScore` field to `PlanAdjustment`.
- `prompts/training-planner.md` — narrow the planner to a specific race distance (e.g., 5 K only).
- `eval-matrix.yaml` — adjust the adherence threshold on the `eval-event` control.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a runner profile → planner assesses baseline, generates a plan, builds workouts, completes within ~3 minutes.
2. Load a skill not in the capability registry → guardrail blocks the dispatch; planner replans using only registered skills.
3. Submit a plan that drifts low in adherence → adherence eval scores the adjustment below threshold; planner replans.
4. Submit a profile where the goal date is too close for a safe build-up → planner emits a `PlanFailed` with a clear reason rather than generating an unsafe plan.

## License

Apache 2.0.
