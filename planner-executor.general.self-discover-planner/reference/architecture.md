# Architecture — self-discover-planner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`SolveEndpoint` is the entry point. A submission writes a `SolveSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `SolveWorkflow` per submission.

The workflow drives the plan-first execution loop: it first asks `PlannerAgent` to compose a `ReasoningPlan` from the module library held in `ModuleRegistry`. That plan is immediately handed to `PlanEvaluatorAgent` before any module runs. Only if the evaluator returns `PlanVerdict.PASS` does the workflow enter the module execution loop, calling `ExecutorAgent` once per step in the accepted plan's `stepOrder`. When all steps are recorded on the `ExecutionLog`, `SynthesiserAgent` combines them into a `TaskAnswer`.

Every transition emits an event on `SolveEntity`; `SolveView` projects those events into the read model the UI streams via SSE.

`RequestSimulator` drips sample tasks for demo purposes. `StuckSolveMonitor` ticks every 30 s to mark long-running `EXECUTING` solves as `STUCK`.

`ModuleRegistry` is a single-instance entity keyed by `"global"`, seeded at startup from `sample-data/modules.jsonl`. The workflow reads it once at the start of each solve and passes the module list as context to the `PlannerAgent`.

## Interaction sequence

The sequence diagram traces the happy path (J1). The first loop — the evaluation gate — can repeat up to twice if the evaluator returns `FAIL`; on a third failure the workflow transitions directly to `failStep`. The second loop — module execution — runs once per step in `stepOrder`, with `ExecutorAgent` receiving the growing `ExecutionLog` at each iteration so it can reference prior step outputs. The diagram collapses the per-step executor calls into a single `loop` block for readability.

## State machine

`SolveEntity` has five states. `PLANNING` is the initial state where the `PlannerAgent` is composing or revising the plan. `EVALUATING` is the gate state — the system stays here until a plan passes the evaluator or the revision budget is exhausted. `EXECUTING` is the module-step loop; `ModuleExecuted` events fire here without changing the status. A solve lands in one of three terminal states: `COMPLETED` (all steps executed and answer synthesised), `FAILED` (plan rejected after maximum revisions, or a step fatally failed), or `STUCK` (no progress for 5 minutes).

## Entity model

`SolveEntity` is the system's source of truth; every workflow transition writes one of eight event types. `ModuleRegistry` carries the static module catalogue; it only emits `RegistrySeeded` once at startup. `RequestQueue` is the audit log of submissions. `SolveView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `composePlanStep` 60 s, `revisePlanStep` 60 s, `evaluatePlanStep` 45 s, `executeModuleStep` 90 s, `synthesiseStep` 60 s, `completeStep` 30 s.
- Revision budget: 2 consecutive `FAIL` verdicts from the evaluator; the third transitions to `failStep`.
- Module loop: steps execute sequentially; each `executeModuleStep` call receives the full `ExecutionLog` so far.
- Idempotency: `(prompt, requestedBy)` over a 10 s window deduplicates `POST /api/solves`.
- Stuck detection: `StuckSolveMonitor` every 30 s; solves `EXECUTING` for > 5 minutes are marked `STUCK`.
- Scrubber determinism: `OutputScrubber.scrub` is pure; same input always yields same scrubbed output.
