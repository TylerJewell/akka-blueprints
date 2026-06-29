# Architecture — sandboxed-analyst-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`AnalysisEndpoint` is the entry point. A dataset upload writes a `DatasetSubmitted` event to `UploadQueue` (event-sourced for audit). An `UploadConsumer` subscribes to that queue and starts an `AnalysisWorkflow` per upload. The workflow drives the planner-executor loop: it asks `AnalystAgent` to plan, then on each iteration asks it to decide. The decision produces a Python script that routes to `SandboxExecutorAgent`, which submits it to the E2B sandbox (or mock executor). Every transition emits an event on `AnalysisJobEntity`; `AnalysisJobView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `DatasetSimulator` drips sample dataset uploads every 90 s for demo purposes; `StaleJobMonitor` ticks every 30 s to mark long-running `EXECUTING` jobs as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every execution step.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → execute → sanitize → record → auto-halt-eval → decide. The two participants (AnalystAgent and SandboxExecutorAgent) are shown separately because they have different capability declarations and prompt contracts.

## State machine

`AnalysisJobEntity` has six states. `PLANNING` is the initial state. `EXECUTING` is the loop state — most events fire here without changing the status. A job lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (analyst exhausted its replan or failure budget), `HALTED` (operator pressed Halt or the auto-safety evaluator flagged an unsafe signal), or `STUCK` (no progress for 5 minutes).

## Entity model

`AnalysisJobEntity` is the system's source of truth; every transition writes one of eleven event types. `SystemControlEntity` carries the operator halt flag. `UploadQueue` is the audit log of dataset submissions. `AnalysisJobView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `executeStep` 120 s (covers E2B sandbox cold-start plus script runtime), `decideStep` 45 s, `composeReportStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(scriptKind, script)`; the fourth becomes `Fail`.
- Idempotency: `(datasetName, uploaderEmail)` over a 10 s window deduplicates `POST /api/jobs`.
- Stuck detection: `StaleJobMonitor` every 30 s; jobs `EXECUTING` for > 5 minutes are marked `STUCK`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
