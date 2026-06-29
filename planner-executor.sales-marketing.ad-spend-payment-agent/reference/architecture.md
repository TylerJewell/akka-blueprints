# Architecture — ad-spend-payment-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`CampaignEndpoint` is the entry point. A brief submission writes a `BriefSubmitted` event to `CampaignQueue` (event-sourced for audit). A `CampaignRequestConsumer` subscribes to that queue and starts a `CampaignWorkflow` per submission. The workflow drives the planner-executor loop: it asks `CampaignPlannerAgent` to plan, then on each iteration asks it to decide. The decision routes to one of two specialist agents — `CopywriterAgent` or `PaymentExecutorAgent`. Payment decisions are additionally gated through `ApprovalEntity`, which pauses the workflow until a human operator approves via the UI. Every transition emits an event on `CampaignEntity`; `CampaignView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `BriefSimulator` drips sample briefs for demo purposes; `StaleCampaignMonitor` ticks every 60 s to mark campaigns that have been `EXECUTING` or `AWAITING_APPROVAL` for more than 10 minutes as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every loop iteration.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle. For a COPYWRITER dispatch the workflow calls `CopywriterAgent` directly; for a PAYMENT dispatch it first pauses at `ApprovalEntity`, waits for the human to act, then calls `PaymentExecutorAgent`. The sanitize step follows every executor call.

## State machine

`CampaignEntity` has seven states. `PLANNING` is the initial state. `EXECUTING` is the normal loop state. `AWAITING_APPROVAL` is the suspended state while a payment decision is pending human sign-off — multiple events can fire here without changing the status. A campaign lands in one of four terminal states: `COMPLETED` (happy path), `FAILED` (planner exhausted its budgets), `HALTED` (operator halt), or `STUCK` (no progress for 10 minutes in either loop state).

## Entity model

`CampaignEntity` is the source of truth with thirteen event types. `ApprovalEntity` is a per-placement subordinate entity whose state machine (PENDING → APPROVED | REJECTED) gates the workflow. `SystemControlEntity` carries the operator halt flag. `CampaignQueue` is the submission audit log. `CampaignView` is the only read-side projection — the UI never queries entities directly.

## Concurrency and timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `approvalGateStep` 600 s, `dispatchStep` 120 s, `decideStep` 45 s, `composeReportStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(executor, task)`; the fourth becomes `Fail`.
- Budget accounting: remaining budget is recomputed from the payment ledger at every `guardrailStep` — no cached total.
- Approval idempotency: `POST /approvals/{placementId}/approve` is safe to call more than once; the entity ignores duplicate grants.
- Stuck detection: `StaleCampaignMonitor` every 60 s; covers both `EXECUTING` and `AWAITING_APPROVAL` to prevent abandoned approval requests from blocking campaigns indefinitely.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching.
