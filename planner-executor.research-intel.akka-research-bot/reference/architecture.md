# Architecture — akka-research-bot

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`JobEndpoint` is the entry point. A submission writes a `JobSubmitted` event to `JobQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ResearchWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to plan, then for each query in the plan it runs the `SearcherAgent`. After all queries complete, the planner assesses the collected evidence; on `Sufficient`, the workflow calls `WriterAgent`. Every transition emits an event on `ResearchJobEntity`; `ResearchJobView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `JobSimulator` drips sample jobs for demo purposes; `StaleJobMonitor` ticks every 30 s to mark long-running `SEARCHING` jobs as `STALE`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every search dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The inner loop is the per-query cycle — for each query: check-halt → guardrail → search → sanitize → record. After the loop the workflow calls the planner for assessment, then the writer. The diagram shows the guardrail and sanitizer as workflow-internal steps; in code they are `queryGuardrailStep` and `sanitizeStep` respectively, wired between the agent calls.

## State machine

`ResearchJobEntity` has seven states. `PLANNING` is the initial state. `SEARCHING` is the query-loop state — most events here do not change status. `WRITING` is a distinct state entered after the planner returns `Sufficient`; the writer operates here. A job lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (replan or revision budget exhausted), `HALTED` (operator paused), or `STALE` (no progress for 5 minutes).

## Entity model

`ResearchJobEntity` is the system's source of truth; twelve event types cover the full lifecycle from creation through every query, revision, and terminal outcome. `SystemControlEntity` carries the operator halt flag. `JobQueue` is the audit log of submissions. `ResearchJobView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `searchStep` 90 s, `assessStep` 45 s, `writeStep` 120 s, `reportGuardrailStep` 30 s.
- Replan budget: 2 consecutive `NeedsMore` returns; the third becomes `Fail`.
- Report revision budget: 2 `REVISE_REPORT` calls; a third `ReportBlocked` becomes `Fail`.
- Idempotency: `(question, requestedBy)` over a 10 s window deduplicates `POST /api/jobs`.
- Stale detection: `StaleJobMonitor` every 30 s; jobs `SEARCHING` for > 5 minutes are marked `STALE`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every query iteration; no caching.
