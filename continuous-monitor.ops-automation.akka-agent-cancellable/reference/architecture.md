# Architecture — activity-interrupt-cancellation

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`TaskPoller` is the heartbeat — a TimedAction that ticks every 20 s and writes simulated `TaskSubmitted` events into `TaskQueue` (event-sourced for audit). A `TaskDispatcher` Consumer subscribes to that queue, registers the task on `ActivityEntity`, and starts one `ActivityWorkflow` per task. The workflow orchestrates a loop: it calls `TaskRunnerAgent` iteratively, recording each `StepResult` as a `StepCompleted` event, until the agent reports `terminal=true` or the entity signals CANCELLING.

`StaleActivityReaper` runs alongside as a second TimedAction, ticking every 5 minutes and emitting `CancellationRequested` on any RUNNING task whose `startedAt` is older than the configured timeout.

## Interaction sequence

The sequence traces the cancel path (J1 + J2 combined). Two events of interest: (1) the `CancellationRequested` event written synchronously by `ActivityEndpoint.cancel`, which transitions the entity to CANCELLING; (2) the workflow detecting CANCELLING at the next `executeStep` boundary, passing `shouldContinue=false` to the agent, and branching to `cleanupStep`.

The `CleanupAgent` call is synchronous within the workflow — the `TaskCancelled` terminal event is not emitted until the cleanup plan is recorded.

## State machine

Six states. The two cancellation paths share the same CANCELLING → CANCELLED arc: operator-initiated cancel sets the entity to CANCELLING via the API; the stale reaper does the same via a direct entity command. Both paths arrive at `cleanupStep` in the workflow.

COMPLETED and FAILED are terminal states reached without passing through CANCELLING. FAILED is emitted when a workflow step fails unrecoverably (e.g., cleanup agent call fails after timeout).

## Entity model

`ActivityEntity` is the source of truth; it emits eight distinct event types covering the full lifecycle including cleanup. `TaskQueue` is the upstream audit log — only `TaskDispatcher` subscribes to it. `ActivityView` projects from `ActivityEntity` for read-path queries and SSE.

## Two-halt defence

For any activity that ends in CANCELLED, the following sequence has occurred:
1. **Operator-regulator-stop** (H1): the `CancellationRequested` event was recorded; the entity entered CANCELLING; the workflow stopped advancing the agent.
2. **Graceful-degradation** (H2): `CleanupAgent` produced a `CleanupPlan`; `CleanupRecorded` was emitted; only then was `TaskCancelled` written.

Neither halt can be bypassed independently. H1 without H2 is prevented by the workflow's mandatory `cleanupStep`. H2 without H1 is not possible because `cleanupStep` is only reachable from the CANCELLING branch.
