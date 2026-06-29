# Architecture — research-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`ResearchEndpoint` is the entry point. A submission writes a `QuerySubmitted` event to `QueryQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ResearchWorkflow` per submission. The workflow drives the planner-executor loop: it asks `ResearchPlannerAgent` to design a search strategy, then on each iteration asks it to decide the next step. The decision routes to either `WebSearchAgent` or `DocumentSearchAgent`. After each search result returns, the workflow's `citationEvalStep` runs `CitationEvaluator` to score the finding before appending it to `ResearchJobEntity`. `ResearchJobView` projects the entity events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `QuerySimulator` drips sample queries for demo purposes; `StaleJobMonitor` ticks every 30 s to mark long-running `SEARCHING` jobs as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI dashboard; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → dispatch → citation-eval → record → decide. The diagram condenses both searchers into a single participant for readability; in code the workflow's `dispatchStep` switches on `SearchDecision.searcher` to call the matching agent.

## State machine

`ResearchJobEntity` has six states. `PLANNING` is the initial state; `JobPlanned` moves the job to `SEARCHING`. `SEARCHING` is the loop state — `FindingRecorded`, `StepRetried`, and `LedgerRevised` events fire here without changing the status. A job lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (planner exhausted its replan or failure budget), `HALTED` (operator pressed Halt), or `STUCK` (no progress for 5 minutes).

## Entity model

`ResearchJobEntity` is the system's source of truth; every transition writes one of nine event types. `SystemControlEntity` carries the operator halt flag. `QueryQueue` is the audit log of submissions. `ResearchJobView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `citationEvalStep` 15 s, `decideStep` 45 s, `composeReportStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(searcher, query)`; the fourth becomes `Fail`.
- Idempotency: `(query, requestedBy)` over a 10 s window deduplicates `POST /api/jobs`.
- Stale detection: `StaleJobMonitor` every 30 s; jobs `SEARCHING` for > 5 minutes are marked `STUCK`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
- Citation evaluator determinism: `CitationEvaluator.evaluate` is pure; same input always yields the same verdict and confidence, keeping `FindingEntry` events deterministic and replayable.
