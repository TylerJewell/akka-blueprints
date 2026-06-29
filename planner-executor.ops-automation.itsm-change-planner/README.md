# Akka Sample: ITSM Change Management Agent

A PlannerAgent analyzes an incoming change request, retrieves similar historical changes, and produces three coordinated plans — implementation, test, and backout — on a change ledger. A Change Advisory Board approval gate (HITL) holds execution until a reviewer signs off. A production-touch guardrail intercepts each implementation step before the executor runs it. If a test step fails, an automatic safety halt triggers the pre-generated backout plan. Demonstrates the **planner-executor** coordination pattern with mandatory human approval and production-safety governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Change requests, historical change records, and CMDB configuration items are seeded as fixtures inside the same Akka service.

## Generate the system

```sh
cp -r ./planner-executor.ops-automation.itsm-change-planner  ~/my-projects/itsm-change-planner
cd ~/my-projects/itsm-change-planner
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that reads a change request, queries historical change fixtures, and produces a `ChangeLedger` holding implementation, test, and backout plans.
- **ExecutorAgent** — AutonomousAgent that executes one implementation step at a time, returning a `StepResult` with evidence of what was done.
- **TestRunnerAgent** — AutonomousAgent that runs the test step corresponding to the just-executed implementation step and returns a `TestResult`.
- **BackoutAgent** — AutonomousAgent that executes backout steps when a test step fails or the operator initiates rollback.
- **ChangeWorkflow** — Workflow driving plan → cab-approval → execute-step → test-step → decide loop, with automatic backout branch and terminal states.
- **ChangeEntity** — EventSourcedEntity holding the change's lifecycle, its `ChangeLedger`, and the running `ExecutionLog`.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag (single instance keyed by `"global"`).
- **ChangeQueue** — EventSourcedEntity audit log of submitted change requests.
- **ChangeView** — View projection used by the UI.
- **ChangeRequestConsumer** — Consumer that subscribes to `ChangeQueue` events and starts a `ChangeWorkflow` per submission.
- **ChangeSimulator** — TimedAction that drips a sample change request every 90 s so the App UI is not empty on first load.
- **StuckChangeMonitor** — TimedAction that marks any change stuck in an execution state past 10 minutes.
- **ChangeEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the change-request prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ChangeRequest` record fields (e.g., add `urgency`).
- `prompts/planner.md` — constrain the planner to a specific CI type (e.g., database-only changes).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-step quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a change request → planner produces a three-part plan, CAB approval unblocks execution, all steps execute and pass tests, change moves to `IMPLEMENTED`.
2. CAB reviewer rejects the plan → change moves to `REJECTED`; no execution steps run.
3. A test step fails → automatic safety halt triggers; backout plan executes; change moves to `ROLLED_BACK`.
4. Submit a change whose implementation step targets a production path not on the allow-list → production-touch guardrail blocks it before the executor runs; planner receives the block and replans.

## License

Apache 2.0.
