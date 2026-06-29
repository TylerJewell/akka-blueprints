# Architecture — akka-memory-first-coding-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`ProjectEndpoint` is the primary entry point. An init submission creates a `Project` record on `ProjectEntity` and starts a `ResearchWorkflow`. An edit submission calls `ProjectEntity.requestEdit`, which emits `EditSessionRequested`; `SessionRequestConsumer` picks that event up and starts an `EditWorkflow`.

`ResearchWorkflow` calls three agents in sequence: `ResearchPlannerAgent` to scan the fixture index, `CodeReaderAgent` in a loop once per selected file, and `MemoryWriterAgent` to synthesise the collected `FileInsight` records into `MemoryBlock` entries and a rewritten system prompt. Both the blocks and the prompt are persisted back onto `ProjectEntity`.

`EditWorkflow` calls `EditPlannerAgent` once to produce a `PatchPlan`, then loops over the plan's `FileEdit` operations. Before each apply it checks the operator halt flag on `SystemControlEntity` and runs `EditGuardrail.vet`. After each apply it runs `FixtureTestRunner`. Both entities — `ProjectEntity` and `EditSessionEntity` — emit events that `ProjectView` projects into the read model consumed by the UI's SSE stream.

`ResearchSimulator` fires once on startup to populate the UI with a sample project. `StaleSessionMonitor` ticks every 30 s to catch sessions stuck in `APPLYING`.

## Interaction sequence

Two sequence diagrams cover the two primary flows. The first traces the `/init` happy path: index scan, per-file read loop, synthesis, and memory persistence. The second traces the `/edit` path including the guardrail rejection branch (J2). Both diagrams condense participants for readability — in code the workflow's step methods drive the actual agent invocations via `forAutonomousAgent(...).runSingleTask(...)`.

## State machine

`EditSessionEntity` has seven states. `PLANNING` is the entry state; the session moves to `APPLYING` when the patch plan is ready. The `APPLYING` state is the loop state — `PatchApplied`, `TestsPassed`, and `PatchBlocked` events fire here without changing status. The session reaches one of five terminal states: `COMPLETED` (all edits applied and tests passed), `TESTS_FAILED` (a test run returned failures), `FAILED` (planner exhausted its revision budget), `HALTED` (operator pressed Halt), or `TIMED_OUT` (no progress for 5 minutes).

`ProjectEntity` has three states: `RESEARCHING` (research workflow in progress), `READY` (memory blocks built), and `FAILED` (research workflow errored).

## Entity model

`ProjectEntity` is the project's source of truth; its events record every phase of the research lifecycle including the `EditSessionRequested` event that triggers the consumer. `EditSessionEntity` holds the full patch history for one edit request. `SystemControlEntity` carries the operator halt flag. `ProjectView` is the only read-side projection — the UI never queries entities directly.

## Concurrency & timeouts

- Per-step timeouts (ResearchWorkflow): `scanStep` 45 s, `readFileStep` 60 s per iteration, `synthesiseStep` 90 s, `persistMemoryStep` 30 s.
- Per-step timeouts (EditWorkflow): `planStep` 60 s, `applyStep` 90 s, `testGateStep` 120 s, `decideStep` 30 s.
- Guardrail-replan budget: at most two consecutive blocked patches before the session fails.
- Test gate is terminal: a single failing test run ends the session; no retry.
- Halt poll: `checkHaltStep` reads `SystemControlEntity.get` synchronously at the top of each loop iteration; no caching.
- Idempotency: `POST /api/projects/init` deduplicates on `(projectPath, requestedBy)` over a 30 s window.
- Stuck detection: sessions in `APPLYING` for more than 5 minutes are marked `TIMED_OUT` by `StaleSessionMonitor`.
- Self-rewrite: `synthesiseStep` loads the current `ProjectEntity.memory.systemPrompt` (if non-empty) and passes it as the overriding system prompt to `MemoryWriterAgent`. The first run uses the static prompt file; subsequent runs use the persisted rewrite.
