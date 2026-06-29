# Akka Sample: Agentic Code Assistant

An agent that plans a code-change task, loops over read-file / edit-file / run-tests cycles, validates the result against the project's test suite, and commits when all tests pass — or stops and reports when a loop-bound or safety halt trips.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The filesystem reads, code edits, and test runs are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.code-assistant-loop  ~/my-projects/code-assistant-loop
cd ~/my-projects/code-assistant-loop
```

(Optional) Edit `SPEC.md` to change the project fixture, model provider, or loop bounds.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that reads the task description and the current file tree, drafts a `ChangePlan { steps, targetFiles, rationale }`, and on each loop tick decides whether to continue, revise, or finalize.
- **ReaderAgent** — AutonomousAgent that returns file contents from seeded fixture files.
- **EditorAgent** — AutonomousAgent that produces a unified diff for a target file.
- **RunnerAgent** — AutonomousAgent that returns simulated test-run output for an allow-listed test command set.
- **SessionWorkflow** — Workflow with a plan → read → edit → run-tests → record → decide loop, replan branch, ci-gate branch, and terminal exit states.
- **SessionEntity** — EventSourcedEntity holding the session lifecycle, the change plan, the edit log, and the final outcome.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **SessionView** — projection used by the UI.
- **SessionEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the simulated task prompts, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Session` record fields (e.g., add `confidenceScore` per change).
- `prompts/planner.md` — scope the planner to a single language or framework.
- `eval-matrix.yaml` — add an `eval-event` control if you want per-edit quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a coding task → planner drafts a change plan, editor patches each file, test runner validates, session ends in `COMMITTED` within ~3 minutes.
2. Propose a file write outside `/workspace/` → the pre-tool guardrail blocks the edit; the planner replans; the session either commits via a different path or fails after the replan budget is exhausted.
3. Tests fail twice in a row → the CI gate trips and the session ends in `FAILED` with a clear failure reason.
4. Click **Halt new edits** while `EXECUTING` → the in-flight edit completes; no further edits dispatch; session ends in `HALTED`.

## License

Apache 2.0.
