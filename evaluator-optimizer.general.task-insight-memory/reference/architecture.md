# Architecture — task-insight-memory

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`MemoryEndpoint` is the entry point for task submissions. It writes a `TaskSubmitted` event to `TaskQueue` (event-sourced for audit). A `TaskConsumer` subscribes to that queue and starts a `MemoryWorkflow` per submission. The workflow has four meaningful steps: it queries `MemoryView` for prior insights of the same task type, calls `ExecutorAgent` to produce a result, calls `EvaluatorAgent` to score that result, and — if the verdict is `VERIFIED` and confidence clears the threshold — runs a deterministic PII sanitize pass before writing the insight to `MemoryEntity`.

`TaskEntity` records every task lifecycle event; `MemoryEntity` holds the accumulated insight store. Both project into views (`TaskView`, `MemoryView`) that the UI streams via SSE.

Two TimedActions run alongside the workflow: `TaskSimulator` drips a sample task every 60 seconds so the UI is not empty when first loaded; `DriftWatcher` ticks every 5 minutes, computes distribution metrics over the full insight store, and emits a `DriftAssessmentRecorded` event when type concentration or average confidence crosses a threshold.

## Interaction sequence

The sequence diagram traces a J1-style verified persistence where a task is submitted, executed with no prior insights (cold start), evaluated as `VERIFIED`, sanitized, and persisted. The `EvaluatorAgent` → sanitize → persist chain is explicit. Each step writes to `TaskEntity` before the next step begins, so the UI's per-task detail view reconstructs the full lifecycle accurately.

## State machine

The task moves through two transient states (`EXECUTING`, `EVALUATED`) and two terminal states (`VERIFIED`, `FAILED`). `PENDING` is the creation state; the workflow immediately transitions the task to `EXECUTING` when `startStep` fires. `EVALUATED` is the moment after the evaluator returns its verdict but before the persist decision is made. The `VERIFIED → [*]` path means an insight was written to the memory store; the `FAILED → [*]` path means the result did not meet the bar and no insight was persisted. The `defaultStepRecovery` failover from `EXECUTING → FAILED` captures irrecoverable agent failures without leaving the task in a hanging state.

## Entity model

`TaskEntity` is the per-task source of truth; it emits six event types covering the full lifecycle. `MemoryEntity` is the long-lived accumulation store; it emits five event types covering the three write paths (verified-experience, correction, demonstration) plus insight supersession and drift assessments. `TaskQueue` is the audit log of submissions. The two views — `TaskView` and `MemoryView` — are the only read-side projections; neither the workflow nor the endpoint queries entities directly.

## Concurrency and timeouts

- Per-step timeout: 60 s for `executeStep` and `evaluateStep`; `retrieveStep` and `sanitizeStep` are in-process and effectively instant.
- Workflow-wide bound: the workflow runs at most one executor call and one evaluator call per task submission; there is no retry loop at the workflow level. Confidence-gating is a one-shot decision.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure ends in `TaskFailed`, not in a hung workflow.
- Idempotency: `MemoryEndpoint.submitTask` deduplicates on `(taskType, description)` over a 10 s window. `DriftWatcher` deduplicates on the 5-minute window timestamp.
- Insight retrieval ordering: `getInsightsByTaskType` orders by `persistedAt DESC`; superseded insights are excluded at the query level so the executor never receives stale or overwritten learnings.
- Correction and demonstration writes bypass the workflow entirely; they write directly to `MemoryEntity` via `MemoryEndpoint`. The `InsightProvenance` enum distinguishes these from `VERIFIED_EXPERIENCE` insights in all views and in the UI's memory panel.
