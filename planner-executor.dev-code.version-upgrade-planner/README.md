# Akka Sample: Airflow Version Upgrade Agent

An UpgradePlannerAgent analyses the current Apache Airflow installation, builds an ordered upgrade plan on a plan ledger, dispatches each phase to one of three executor agents — CompatibilityChecker, TestRunner, MigrationApplier — tracks progress on a progress ledger, and waits for human approval before applying destructive migration steps. A CI gate validates the test suite after every applied phase. Demonstrates the **planner-executor** coordination pattern with a human-in-the-loop approval gate and a CI test gate.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Airflow environment introspection, compatibility checks, test execution, and migration application are all simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.version-upgrade-planner  ~/my-projects/version-upgrade-planner
cd ~/my-projects/version-upgrade-planner
```

(Optional) Edit `SPEC.md` to change the target Airflow version, the approval-gate reviewer role, or any executor agent's behavior.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **UpgradePlannerAgent** — AutonomousAgent that maintains a plan ledger (target version, current version, ordered phases, blocked phases) and a progress ledger (per-phase attempt count, verdict, migration result). Decides which phase runs next. Replans when a phase fails.
- **CompatibilityCheckerAgent** — AutonomousAgent that inspects provider and DAG compatibility against the target version using seeded fixture data.
- **TestRunnerAgent** — AutonomousAgent that executes the Airflow test suite (simulated) and returns a structured test report.
- **MigrationApplierAgent** — AutonomousAgent that applies a single migration phase from seeded migration scripts and returns the outcome.
- **UpgradeWorkflow** — Workflow with a plan → approval-gate → execute → record → ci-validate → decide loop, replan branch, and terminal exit states.
- **UpgradeJobEntity** — EventSourcedEntity holding the upgrade job lifecycle, both ledgers, and the final upgrade report.
- **ApprovalEntity** — EventSourcedEntity holding the pending approval request and the human reviewer's decision.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **UpgradeJobView** — projection used by the UI.
- **UpgradeJobEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample Airflow versions the simulator uses, or remove the simulator.
- `SPEC.md §5` — adjust the `UpgradeJob` record fields (e.g., add `rollbackPlanId`).
- `prompts/upgrade-planner.md` — narrow the planner to a specific migration path (e.g., 2.x → 3.x only).
- `eval-matrix.yaml` — add a `sanitizer` control if you want credential scrubbing on migration script output.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an upgrade job → planner plans phases, the approval gate fires, a reviewer approves, execution proceeds, tests pass the CI gate, job completes.
2. A reviewer rejects the approval request → the workflow records the rejection on the progress ledger, marks the phase blocked, and the planner replans or fails gracefully.
3. Submit a job and click **Halt new phases** while a phase is executing → the in-flight phase finishes; no further phases dispatch; the job moves to `HALTED`.
4. The test runner returns a failing test report → the CI gate blocks the next phase from dispatching, and the planner is asked to replan or abort.

## License

Apache 2.0.
