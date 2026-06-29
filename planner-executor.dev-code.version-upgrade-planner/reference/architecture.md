# Architecture — version-upgrade-planner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`UpgradeJobEndpoint` is the entry point for job submissions and approval decisions. A submission writes an `UpgradeSubmitted` event to `UpgradeRequestQueue` (event-sourced for audit). `UpgradeRequestConsumer` subscribes to that queue and starts an `UpgradeWorkflow` per job. The workflow drives the planner-executor loop: it asks `UpgradePlannerAgent` to plan the upgrade phases, then on each iteration asks it to decide. The decision routes to one of three executor agents — `CompatibilityCheckerAgent`, `TestRunnerAgent`, or `MigrationApplierAgent`. Every transition emits an event on `UpgradeJobEntity`; `UpgradeJobView` projects those events into the read model the UI streams via SSE.

When a migration phase requires approval, the workflow writes a pending `ApprovalRequest` to `ApprovalEntity` and suspends until a reviewer calls the approve or reject endpoint. `SystemControlEntity` is a single-instance entity keyed by `"global"` — operators flip its halt flag from the UI; the workflow polls the flag before every phase dispatch.

Two TimedActions run alongside the core loop: `UpgradeRequestSimulator` drips sample jobs for demo purposes; `StuckJobMonitor` ticks every 60 s to mark long-running jobs as `STUCK`.

## Interaction sequence

The sequence diagram traces the happy path (J1), including the approval gate. The approval gate is the distinguishing feature of this blueprint: the workflow writes to `ApprovalEntity`, emits `ApprovalRequested` on `UpgradeJobEntity` (status → `AWAITING_APPROVAL`), and then polls `ApprovalEntity` until a reviewer resolves it. Once resolved, the workflow emits `ApprovalGranted` (status → `EXECUTING`) and continues to the executor.

The CI gate (after every `MIGRATION` phase) is condensed into the `ciGateStep` in the loop body; the diagram shows it as part of the executor interaction. In code, `ciGateStep` calls `TestRunnerAgent.RUN_TESTS` and, on test failures, appends a `CiGateFailed` progress entry before returning to `plannerDecideStep`.

## State machine

`UpgradeJobEntity` has seven states. `PLANNING` is the initial state. `EXECUTING` is the main loop state. `AWAITING_APPROVAL` is a blocking intermediate state that the workflow enters before each migration phase; the job returns to `EXECUTING` after the reviewer resolves the approval request. Terminal states: `COMPLETED` (all phases passed), `FAILED` (planner exhausted its replan or failure budget), `HALTED` (operator pressed Halt), `STUCK` (no progress for 10 minutes).

The `AWAITING_APPROVAL` → `EXECUTING` transition fires on both approval and rejection — in the rejection case, the workflow records `PhaseBlocked` and the planner can choose to replan.

## Entity model

`UpgradeJobEntity` is the source of truth for the upgrade job lifecycle; it emits fourteen event types covering every significant state change. `ApprovalEntity` is keyed per approval request (one per migration phase that requires approval); its two events — `ApprovalCreated` and `ApprovalResolved` — are the contract between the reviewer's HTTP call and the suspended workflow step. `SystemControlEntity` carries the operator halt flag. `UpgradeRequestQueue` is the submission audit log. `UpgradeJobView` is the only read-side projection — the UI never queries entities directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 90 s, `plannerDecideStep` 60 s, `approvalGateStep` 3600 s, `dispatchStep` 180 s, `ciGateStep` 120 s, `composeReportStep` 60 s.
- Approval gate: bounded polling with back-off inside the 3600 s step timeout; `StuckJobMonitor` catches jobs that exceed 10 minutes of combined wall-clock time across `EXECUTING` and `AWAITING_APPROVAL`.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive attempts on the same `(executor, phaseId)`; the fourth becomes `Fail`.
- CI gate: non-blocking on first failure — the gate informs the planner rather than aborting. The planner may add a rollback phase.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
