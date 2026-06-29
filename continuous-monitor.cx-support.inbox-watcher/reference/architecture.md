# Architecture — inbox-watcher

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`InboxPoller` is the heartbeat — a TimedAction that ticks every 15 s and writes simulated `MessageReceived` events into `MessageQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, and emits `MessageSanitized` on the per-message `InboxMessageEntity`. That sanitized event starts an `InboxWorkflow` instance, which orchestrates: classify → (optionally draft) → pause for human approval → finalise to SENT/DISMISSED.

`EvalRunner` runs alongside as a second TimedAction, ticking every 30 minutes and scoring sampled sent messages.

## Interaction sequence

The sequence traces the happy path (J1 + J2 combined). Note two distinct pauses: (1) the `PiiSanitizer` Consumer's natural subscription lag (sub-second), and (2) the workflow's explicit AWAITING_APPROVAL pause (open-ended; the workflow polls the entity every 5 s for a human decision).

The `Note over E,U: Workflow pauses` block marks the second pause — this is where the human-in-loop gate lives, and it is the system's primary safety mechanism.

## State machine

Nine states. The interesting branches:

- After CLASSIFIED, the classification value branches the flow: AUTO_DRAFTED proceeds to DRAFTED; ESCALATE jumps to ESCALATED terminal; IGNORE jumps to DISMISSED terminal.
- AWAITING_APPROVAL has two outbound transitions: human Approve → SENT terminal; human Reject → DISMISSED terminal. There is no auto-timeout — drafts wait indefinitely. (Deployers wanting an SLA can add a TimedAction that escalates stale drafts after N hours.)

## Entity model

`InboxMessageEntity` is the source of truth; it emits ten distinct event types covering the full lifecycle including eval. `MessageQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it.

## Defence-in-depth governance flow

For any reply that goes out, the message passed through:
1. **PII sanitizer** — never let the model see identifiers.
2. **InboxFilterAgent** — typed classifier defaults to ESCALATE under uncertainty.
3. **DraftReplyAgent** — drafts but cannot send.
4. **HITL gate** — human must Approve.
5. **before-tool-call guardrail** — if a future change wires a real send tool, the guardrail double-checks the approval state.

Each step is independent. Removing any one of them does not silently downgrade the system; it explicitly opens a vulnerability that the eval-periodic sampler is likely to catch.
