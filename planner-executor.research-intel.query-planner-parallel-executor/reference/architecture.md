# Architecture — query-planner-parallel-executor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`QueryEndpoint` is the entry point. A submission writes a `QuerySubmitted` event to `QueryQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `QueryWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to decompose the question into sub-queries, runs a plan-quality evaluation, then fans out the sub-queries in parallel to the three retrieval executors — `CorpusSearchExecutor`, `WebLookupExecutor`, and `KnowledgeBaseExecutor`. After each round, the Planner evaluates coverage and decides whether to continue, request a follow-up round, or proceed to synthesis. `SynthesisAgent` merges the collected results into a `ResearchAnswer`. Every transition emits an event on `QuerySessionEntity`; `SessionView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `QuerySimulator` drips sample research questions every 90 seconds; `StaleSessionMonitor` marks long-running sessions as `STALE` every 30 seconds.

`SystemControlEntity` is a single-instance entity keyed by `"global"`. Operators flip its halt flag from the UI; the workflow reads it before each round's fan-out.

## Interaction sequence

The sequence diagram traces the happy path (J1). The planner-executor loop runs in the centre: each round starts with a halt check, fans out all sub-queries in parallel (shown as a `par` block), collects and scrubs results, and ends with the Planner's coverage evaluation. A `Sufficient` verdict exits the loop into synthesis. The diagram condenses the three executor types into a single participant for readability; in code `QueryWorkflow`'s `fanOutStep` dispatches each sub-query to its strategy-matched executor via `switch` on `SubQuery.strategy`.

## State machine

`QuerySessionEntity` has eight states. `PLANNING` is the initial state where the Planner decomposes the question. `EVALUATING` is the plan-quality check state — the session can loop here across revisions before being allowed to proceed. `EXECUTING` is the fan-out state; most sub-query events fire here without changing the status. `SYNTHESIZING` is a brief terminal-approach state while `SynthesisAgent` runs. Terminal states are `COMPLETED` (happy path), `FAILED` (plan never passed quality or coverage exhausted), `HALTED` (operator or auto-safety), and `STALE` (no progress for 5 minutes).

## Entity model

`QuerySessionEntity` is the system's source of truth; every transition writes one of thirteen event types. `SystemControlEntity` carries the operator halt flag. `QueryQueue` is the audit log of submissions. `SessionView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- **Fan-out concurrency:** all sub-queries in a round are dispatched via `CompletableFuture.allOf`; they run concurrently and the `fanOutStep` waits for all to finish (or time out).
- **Per-step timeouts:** `planStep` 60 s, `evalStep` 30 s, `fanOutStep` 180 s (the full round), `coverageStep` 45 s, `synthesisStep` 90 s.
- **Round budget:** three rounds maximum. A third `NeedsFollowup` verdict is treated as `Fail`.
- **Plan revision budget:** two consecutive low-quality plan evaluations. The third triggers `SessionFailed`.
- **Guardrail is per-sub-query:** each sub-query in a round is vetted individually before its executor call is submitted to the concurrent fan-out. A blocked sub-query does not prevent the others from running.
- **Stale detection:** `StaleSessionMonitor` every 30 s; sessions `EXECUTING` for > 5 minutes are marked `STALE`.
- **Sanitizer runs post-collection:** `SecretScrubber.scrub` runs in `collectStep` on each result's content before `SubQueryRecorded` events are emitted and before results are passed to the Planner.
