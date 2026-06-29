# Architecture — supplier-comms-monitor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`PoOrderPoller` is the heartbeat — a TimedAction that ticks every 20 s and writes simulated `PoFeedReceived` events into `PoFeedQueue` (event-sourced for audit). `PoFeedConsumer` subscribes to that queue, registers each PO on `PurchaseOrderEntity`, and starts a `PoOrderWorkflow` instance per PO. The workflow orchestrates: risk-assess → (optionally draft outreach) → guardrail check → pause for buyer approval if material → finalise to OUTREACH_SENT or PROCUREMENT_REVIEW or ON_TRACK_CONFIRMED.

`AccuracyEvalRunner` runs alongside as a second TimedAction, ticking every 60 minutes and measuring the accuracy of risk predictions against actual delivery outcomes on closed POs.

## Interaction sequence

The sequence traces the AT_RISK happy path (J1 + J2 combined). Two distinct pauses exist: (1) the `PoFeedConsumer` Consumer's natural subscription lag (sub-second), and (2) the workflow's explicit AWAITING_BUYER_APPROVAL pause (open-ended; the workflow polls the entity every 5 s for a buyer decision).

The `Note over E,U: Workflow pauses` block marks the second pause — this is where the buyer HITL gate lives, and it is the system's primary control for preventing unauthorised external communications.

## State machine

Eight states (plus initial and terminal). Notable branches:

- After RISK_ASSESSED, the tier branches the flow: ON_TRACK proceeds directly to ON_TRACK_CONFIRMED terminal; AT_RISK and CRITICAL both proceed to OUTREACH_DRAFTED, from where `requiresBuyerApproval` governs the next branch.
- AWAITING_BUYER_APPROVAL has two outbound transitions: buyer Approve → OUTREACH_SENT terminal; buyer Escalate → PROCUREMENT_REVIEW terminal. There is no auto-timeout — POs await buyer action indefinitely. (Deployers wanting an SLA can add a TimedAction that escalates stale approvals after N hours.)

## Entity model

`PurchaseOrderEntity` is the source of truth; it emits ten distinct event types covering the full lifecycle including accuracy eval. `PoFeedQueue` is the upstream audit log — only `PoFeedConsumer` subscribes to it.

## Defence-in-depth governance flow

For any outreach message that goes out via the HITL path:
1. **DeliveryRiskAgent** — typed classifier defaults to AT_RISK under uncertainty.
2. **SupplierOutreachAgent** — drafts but cannot send; flags material changes as requiring buyer approval.
3. **HITL gate** — buyer must Approve for any material outreach.
4. **before-tool-call guardrail** — the `sendSupplierEmail` tool's pre-call hook re-reads entity state to confirm send eligibility before the simulated stub executes.

Each step is independent. Removing any one of them does not silently change system behaviour; it explicitly removes a layer that the accuracy eval monitor is likely to flag over time.
