# Architecture — itsm-change-planner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`ChangeEndpoint` is the entry point. A submission writes a `ChangeSubmitted` event to `ChangeQueue` (event-sourced for audit). A `ChangeRequestConsumer` subscribes to that queue and starts a `ChangeWorkflow` per submission. The workflow drives the full change lifecycle: it asks `PlannerAgent` to produce a three-part `ChangeLedger`, then parks in a CAB approval poll. Once a reviewer approves via the endpoint, the workflow enters the executor loop — guardrail-vetting each implementation step, dispatching it to `ExecutorAgent`, then dispatching the corresponding test step to `TestRunnerAgent`. On a test failure, `BackoutAgent` handles the reverse sequence.

Every transition emits an event on `ChangeEntity`; `ChangeView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `ChangeSimulator` drips sample change requests for demo purposes; `StuckChangeMonitor` ticks every 60 s to mark long-running changes as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the App UI; the workflow polls the flag before every implementation step.

## Interaction sequence

The sequence diagram traces the happy path (J1). The CAB approval poll in the middle shows the workflow blocking until a reviewer acts. The executor loop below it — check-halt → guardrail → execute-step → test-step → record-step — repeats once per implementation step in the `ChangeLedger`. The diagram shows a single representative iteration; in code the workflow iterates over the `implementationPlan` list.

## State machine

`ChangeEntity` has eleven states. `PLANNING` is the initial state. `AWAITING_CAB` is the hold point where the CAB approval gate parks the change. `APPROVED` is a transient state entered immediately on approval, before the executor loop begins. `EXECUTING` is the loop state — most events fire here without changing the status. `ROLLING_BACK` is entered when a test step fails and the backout branch begins. Terminal states: `IMPLEMENTED` (all steps passed their tests), `ROLLED_BACK` (backout completed after a test failure), `REJECTED` (CAB said no), `FAILED` (non-recoverable error in workflow), `HALTED` (operator halt), `STUCK` (no progress for 10 minutes).

## Entity model

`ChangeEntity` is the system's source of truth; every transition writes one of fourteen event types. `SystemControlEntity` carries the operator halt flag. `ChangeQueue` is the audit log of submissions. `ChangeView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 90 s, `executeStepStep` 120 s, `testStepStep` 120 s, `backoutStep` 120 s per backout step.
- CAB poll: short sleep-and-poll with no agent timeout; the workflow waits indefinitely until a CAB decision arrives (or `StuckChangeMonitor` marks the change `STUCK` after 10 minutes of `AWAITING_CAB` with no decision).
- Replan budget: at most 2 consecutive `REVISE_STEP` calls for the same step; a third rejection transitions the change to `FAILED`.
- Halt poll: synchronous read of `SystemControlEntity` before every `guardrailStep`; no caching.
- Idempotency: `(summary, ciName, requestedBy)` over a 30 s window deduplicates `POST /api/changes`.
- Backout ordering: `BackoutAgent` receives steps in descending sequence order, ensuring partial rollbacks undo work correctly.
- Stuck detection: `StuckChangeMonitor` every 60 s; changes in `EXECUTING` or `ROLLING_BACK` for > 10 minutes are marked `STUCK`.
