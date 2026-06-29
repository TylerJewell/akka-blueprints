# Architecture — code-assistant-loop

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative shown beside each diagram.

## Component graph

`SessionEndpoint` is the entry point. A submission writes a `TaskSubmitted` event to `TaskQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `SessionWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to draft a change plan, then on each iteration asks it to decide the next action. The decision routes to one of three specialist agents — `ReaderAgent`, `EditorAgent`, `RunnerAgent`. Every transition emits an event on `SessionEntity`; `SessionView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `TaskSimulator` drips sample coding tasks for demo purposes; `StuckSessionMonitor` ticks every 30 s to mark long-running `EXECUTING` sessions as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → dispatch → record → ci-gate → auto-halt-eval → decide. The diagram condenses the three agents into a single participant for readability; in code the workflow's `dispatchStep` switches on `ActionDecision.actionKind` to call the matching agent.

## State machine

`SessionEntity` has six states. `PLANNING` is the initial state. `EXECUTING` is the loop state — most events fire here without changing the status. A session lands in one of four terminal states: `COMMITTED` (all edits applied and tests pass), `FAILED` (planner exhausted its replan or CI-failure budget), `HALTED` (operator pressed Halt or the automatic loop-bound halt triggered), or `STUCK` (no progress for 5 minutes).

## Entity model

`SessionEntity` is the system's source of truth; every transition writes one of twelve event types. `SystemControlEntity` carries the operator halt flag. `TaskQueue` is the audit log of submissions. `SessionView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `commitStep` 60 s.
- Replan budget: 2 consecutive `Replan` decisions without an `OK` edit; the third becomes `Fail`.
- CI failure budget: 3 consecutive `CiGateFailed` entries; the fourth triggers `Fail`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
- Idempotency: `(prompt, requestedBy)` over a 10 s window deduplicates `POST /api/sessions`.
- Stuck detection: `StuckSessionMonitor` every 30 s; sessions `EXECUTING` for > 5 minutes are marked `STUCK`.
- CiGate determinism: `CiGate.evaluate` is pure and never calls the LLM. The same test-output string always yields the same `CiVerdict`, keeping `EditEntry` events deterministic and replayable.
