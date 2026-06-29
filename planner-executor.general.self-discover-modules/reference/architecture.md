# Architecture — self-discover-modules

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`TaskEndpoint` is the entry point. A submission writes a `TaskSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `TaskWorkflow` per submission. The workflow drives the self-discover loop in two phases. In the structuring phase it calls `StructurerAgent` to select modules, compose a `ReasoningStructure`, and optionally revise it. Between those calls, `StructureEvaluator` scores the structure and the workflow gates entry into the solving phase. In the solving phase, the workflow calls `SolverAgent` once per module step, checking the operator halt flag before each call. Every transition emits an event on `TaskEntity`; `TaskView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `RequestSimulator` drips sample tasks for demo purposes; `StuckTaskMonitor` ticks every 30 s to mark long-running tasks as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls it before every solver step.

## Interaction sequence

The sequence diagram traces the happy path (J1). The structuring phase in the upper half covers module selection, structure composition, and the eval gate. The solving loop in the lower half iterates once per `ModuleStep` — each iteration runs check-halt → execute-step → record. The diagram shows `StructurerAgent` and `SolverAgent` as distinct participants to make the phase boundary explicit.

## State machine

`TaskEntity` has six states. `STRUCTURING` is the initial state — both the first composition and any revisions happen here. `SOLVING` is the loop state — `StepExecuted` events fire here without changing the status. A task lands in one of four terminal states: `COMPLETED` (all steps ran and the answer was composed), `FAILED` (eval exhausted the revision budget, or the orchestrator emitted a terminal failure), `HALTED` (operator pressed Halt during solving), or `STUCK` (no progress for 5 minutes in either active state).

## Entity model

`TaskEntity` is the system's source of truth. The Structurer-phase events (`StructureComposed`, `StructureRejected`, `StructureApproved`, `StructureRevised`) track the planning side; the Solver-phase events (`StepExecuted`, `TaskCompleted`, `TaskFailed`, `TaskHaltedOperator`) track execution. `SystemControlEntity` carries the operator halt flag. `RequestQueue` is the audit log of submissions. `TaskView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `selectStep` 45 s, `composeStep` 60 s, `reviseStep` 60 s, `executeStepN` 90 s, `composeAnswerStep` 60 s.
- Revision budget: 2 consecutive eval rejections; the third triggers `failStep`.
- Step iteration: halt is checked before each `executeStepN` so the in-flight step always finishes before the workflow exits.
- Idempotency: `(prompt, requestedBy)` over a 10 s window deduplicates `POST /api/tasks`.
- Stuck detection: `StuckTaskMonitor` every 30 s; tasks in `STRUCTURING` or `SOLVING` for > 5 minutes are marked `STUCK`.
- Evaluator determinism: `StructureEvaluator.evaluate` is pure; the same structure always yields the same score, keeping events deterministic and replayable.
