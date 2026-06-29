# Architecture — rewoo-fixed-plan

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`QueryEndpoint` is the entry point. A submission writes a `QuerySubmitted` event to `RequestQueue` (event-sourced for audit). `QueryRequestConsumer` subscribes and starts a `QueryWorkflow` per submission.

The workflow drives the ReWOO pipeline in three distinct phases. First, it calls `PlannerAgent` once to produce the full `QueryPlan`. Second, it iterates over the plan steps in index order — for each step it resolves `#En` references using already-filled results, runs `ToolCallGuardrail` on the resolved input, dispatches to `WorkerAgent`, scrubs the result, and records a `StepRecord` on `QueryEntity`. Third, when all steps are complete it calls `SolverAgent` to compose the `SolvedAnswer`.

`QueryView` projects `QueryEntity` events into the read model the UI streams via SSE.

`SystemControlEntity` (keyed `"global"`) holds the operator halt flag. The workflow polls it before each step. Two TimedActions sit alongside: `QuerySimulator` drips sample queries every 90 s; `StuckQueryMonitor` ticks every 30 s to mark long-running queries as `STUCK`.

## Interaction sequence

The sequence diagram traces the J1 happy path. The planning phase is a single round-trip to `PlannerAgent`. The execution loop iterates once per step — halt-check, reference resolution, guardrail, Worker call, sanitize, record. After the last step the workflow calls `SolverAgent` once to produce the final answer.

The three agent participants (Planner, Worker, Solver) are distinct — this is the key structural difference from loop-driven patterns where a single orchestrator agent decides on every iteration.

## State machine

`QueryEntity` has six states. `PLANNING` is the initial state while the `PlannerAgent` is writing the plan. `EXECUTING` spans all Worker step calls — most events (`StepStarted`, `StepCompleted`) fire here without changing the status; `QuerySolved` fires at the end and transitions to `COMPLETED`. A blocked step (`StepBlocked`) transitions directly to `FAILED` — there is no replan budget in this pattern. An operator halt lands the query in `HALTED`. A timeout lands it in `STUCK`.

## Entity model

`QueryEntity` is the write-side source of truth; it accumulates one `StepCompleted` event per plan step, each carrying the `StepRecord` with the scrubbed result. The fully-filled plan is reconstructable by replaying all `StepCompleted` events over the `QueryPlanned` base. `SystemControlEntity` holds the operator halt flag. `RequestQueue` is the submission audit log. `QueryView` is the only read-side projection.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `resolveStep` 15 s, `guardrailStep` 10 s, `executeStep` 90 s, `solveStep` 60 s.
- Fixed-plan constraint: the `QueryPlan` is immutable once emitted by `QueryPlanned`. No replan branch exists.
- Guardrail is terminal on reject: `StepBlocked` → `QueryFailed`. One blocking rejection ends the query.
- Halt poll: synchronous read of `SystemControlEntity` before each step; no caching.
- Stuck detection: `StuckQueryMonitor` every 30 s; queries in `EXECUTING` for > 5 minutes receive `QueryFailedTimeout`.
- Idempotency: `(query, requestedBy)` over a 10 s window deduplicates `POST /api/queries`.
