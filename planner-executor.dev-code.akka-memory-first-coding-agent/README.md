# Akka Sample: Code Memory-First Coding Agent

A coding agent that deep-researches a local codebase on first run, writes persistent memory blocks about the project, rewrites its own system prompt from those blocks, and then assists with code edits — keeping every file-write and code-execution step gated by a guardrail, passing all changes through a test gate, and exposing an operator halt path for destructive operations.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The codebase being researched, file-write operations, and test execution are all simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.akka-memory-first-coding-agent  ~/my-projects/akka-memory-first-coding-agent
cd ~/my-projects/akka-memory-first-coding-agent
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchPlannerAgent** — AutonomousAgent that receives the `/init` trigger, scans the codebase index, and produces a `ResearchPlan` — an ordered list of files and questions to answer.
- **CodeReaderAgent** — AutonomousAgent that reads fixture file excerpts and answers questions about them. Produces `FileInsight` records.
- **MemoryWriterAgent** — AutonomousAgent that synthesises `FileInsight` records into named `MemoryBlock` entries and drafts the rewritten system prompt.
- **EditPlannerAgent** — AutonomousAgent that turns an edit request into a `PatchPlan` — an ordered list of `FileEdit` operations.
- **EditExecutorAgent** — AutonomousAgent that applies individual `FileEdit` operations and returns a `PatchResult`.
- **ResearchWorkflow** — Workflow that drives the `/init` path: plan → read-files loop → write-memory → rewrite-prompt.
- **EditWorkflow** — Workflow that drives the `/edit` path: plan-edits → guardrail-gate → apply-edit → test-gate → record-result → decide loop.
- **ProjectEntity** — EventSourcedEntity holding the project's memory blocks, the current system prompt, and lifecycle state.
- **EditSessionEntity** — EventSourcedEntity holding one edit session's patch plan, applied edits, test results, and halt state.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag (single instance keyed by `"global"`).
- **ProjectView** — View projection used by the UI to list memory blocks and edit sessions.
- **SessionRequestConsumer** — Consumer that subscribes to `ProjectEntity` events and starts an `EditWorkflow` per session request.
- **ResearchSimulator** — TimedAction that triggers a sample `/init` project on startup so the App UI is not empty when first loaded.
- **StaleSessionMonitor** — TimedAction that marks any edit session stuck in `APPLYING` past 5 minutes as `TIMED_OUT`.
- **ProjectEndpoint** — REST + SSE endpoint for `/api/projects/*` and `/api/sessions/*`.
- **AppEndpoint** — Serves the embedded UI and `/api/metadata/*`.

## Customise before generating

- `SPEC.md §3` — change the codebase fixtures or add more file types to the seeded project.
- `SPEC.md §5` — adjust the `MemoryBlock` record (e.g., add a `confidence` score).
- `prompts/research-planner.md` — narrow the research planner to a specific language or framework.
- `eval-matrix.yaml` — add an `eval-event` control if you want per-edit quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Run `/init` on a seeded project → memory blocks are written, system prompt is rewritten, UI shows the blocks and prompt within ~3 minutes.
2. Submit an edit request that would write outside the allowed path scope → guardrail blocks the operation; the session records a `PatchBlocked` entry and the planner revises.
3. An edit session produces a patch; the test-gate step runs the fixture test suite; on failure the session ends in `TESTS_FAILED` without applying the patch to the project's memory.
4. Click **Halt** while a destructive edit is `APPLYING` → the in-flight apply finishes; no further edits run; the session moves to `HALTED`.

## License

Apache 2.0.
