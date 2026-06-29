# Akka Sample: Generalist Multi-Agent

An Orchestrator plans a task, dispatches subtasks to four specialist agents — WebSurfer, FileSurfer, Coder, ComputerTerminal — tracks progress on a task ledger plus a progress ledger, and replans when a step fails. Demonstrates the **planner-executor** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The four specialist surfaces — web, file, code execution, terminal — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.general.orchestrator-web-file-code  ~/my-projects/orchestrator-web-file-code
cd ~/my-projects/orchestrator-web-file-code
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OrchestratorAgent** — AutonomousAgent that maintains a task ledger (facts, guesses, plan, dispatch decisions) and a progress ledger (per-subtask attempts, verdicts, blockers). Decides who runs the next subtask. Replans on three consecutive failures.
- **WebSurferAgent** — AutonomousAgent that answers web-search queries from seeded fixtures.
- **FileSurferAgent** — AutonomousAgent that inspects file content from seeded fixtures.
- **CoderAgent** — AutonomousAgent that drafts or edits code snippets.
- **TerminalAgent** — AutonomousAgent that returns simulated command output for a small allow-listed command set.
- **TaskWorkflow** — Workflow with a plan → dispatch → execute → record → decide loop, replan branch, and terminal exit states.
- **TaskEntity** — EventSourcedEntity holding the task lifecycle, both ledgers, and the final result.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **TaskView** — projection used by the UI.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Task` record fields (e.g., add `confidenceScore`).
- `prompts/orchestrator.md` — narrow the orchestrator to a single domain (e.g., research-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-subtask quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → orchestrator plans, dispatches to specialists, completes within ~3 minutes.
2. Inject a tool-policy violation → guardrail blocks the offending subtask; orchestrator records the block on the progress ledger and replans.
3. Trigger the operator halt → no new subtasks dispatch; in-flight subtasks finish; task moves to `HALTED`.
4. A subtask result containing a secret-shaped string is scrubbed before it reaches the orchestrator's next-step prompt.

## License

Apache 2.0.
