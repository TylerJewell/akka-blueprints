# Architecture — campaign-optimizer-loop

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`CampaignEndpoint` is the entry point. A submission writes a `CampaignSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `CampaignWorkflow` per submission. The workflow drives the planner-executor loop: it asks `PlannerAgent` to plan the campaign, then on each iteration asks it to decide the next step. The decision routes to one of four specialist agents — `CopyWriterAgent`, `AudienceSegmenterAgent`, `AssetPublisherAgent`, `PerformanceAnalystAgent`. Every transition emits an event on `CampaignEntity`; `CampaignView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside the main loop. `CampaignSimulator` drips sample campaign briefs every 120 s. `PerformanceMonitor` ticks every 60 s to check KPIs for campaigns in `LIVE` status and fires a performance alert when metrics miss their thresholds.

`ApprovalEntity` is keyed by `campaignId`. The workflow blocks on it after the Planner emits `SubmitForApproval`; the marketer writes their decision via `POST /api/campaigns/{id}/approve` or `reject`.

## Interaction sequence

The sequence diagram traces the happy path (J1) including the approval gate. The loop in the middle is the planner-executor cycle — each iteration runs propose → guardrail (COPY only) → dispatch → record → decide. After the Planner emits `SubmitForApproval`, the workflow moves to `approvalGateStep`, which blocks on `ApprovalEntity`. The marketer's approval unblocks the workflow; the campaign moves to `LIVE` and the report is composed. The four specialists are condensed into a single participant for readability; in code the workflow's `dispatchStep` switches on `DispatchDecision.specialist`.

## State machine

`CampaignEntity` has seven states. `PLANNING` is the initial state. `EXECUTING` is the main loop state — most step events fire here without changing status. `AWAITING_APPROVAL` is the gate state; it is the only state from which the campaign can transition to either `LIVE` (on approval) or `REJECTED` (on denial). `LIVE` is the post-approval production state; it re-enters the specialist loop if the performance monitor fires an alert. Terminal states: `COMPLETED` (all KPIs met), `FAILED` (planner exhausted its budgets), `REJECTED` (marketer denied).

## Entity model

`CampaignEntity` is the source of truth; it emits thirteen event types across the campaign lifecycle. `ApprovalEntity` carries the marketer's decision, keyed per campaign. `RequestQueue` is the audit log of all submitted briefs. `CampaignView` is the only read-side projection — the UI never queries the entity directly; it reads `CampaignRow` objects from the view and fetches the full campaign by id on click.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `composeReportStep` 60 s, `approvalGateStep` 1800 s.
- Approval timeout: after 30 minutes in `approvalGateStep` without a decision, the workflow emits `CampaignFailed` with `failureReason = "approval timeout"`.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(specialist, step)`; the fourth becomes `Fail`.
- Performance monitor: idempotent per 60 s window; fires `PerformanceAlertFired` at most once per tick per campaign.
- Guardrail scope: `CopyGuardrail.check` intercepts only `StepResult` from `CopyWriterAgent`; all other specialist results bypass it.
- Idempotency: `(goal, requestedBy)` over a 10 s window deduplicates `POST /api/campaigns`.
