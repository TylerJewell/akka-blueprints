# Architecture — project-tracker

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`PlannerPoller` is the heartbeat — a TimedAction that ticks every 20 s and writes simulated `BoardEventReceived` events into `BoardEventQueue` (event-sourced for audit). A `BoardSyncConsumer` Consumer subscribes to that queue, normalises each raw board event into typed domain records, and emits `TaskCreated` on the per-task `PlannerTaskEntity`. That create event starts a `PlannerWorkflow` instance, which orchestrates: recommend owner → guardrail check → dispatch assign tool → monitor for completion.

`StaleTaskChecker` runs independently every 5 minutes, queries `BoardView` for overdue tasks, calls `NudgeAgent` to draft messages, runs them through `GuardrailChecker`, and dispatches the nudge (or escalation) to `PlannerTaskEntity`.

`EvalRunner` runs every 60 minutes, sampling COMPLETED tasks and scoring the assignment quality.

## Interaction sequence

The sequence traces the happy path for a new task arriving (J1) followed by an overdue nudge (J2). Note the two distinct guardrail checks: one inside `PlannerWorkflow` before `assignTask`, and one inside `StaleTaskChecker` before `sendNudge`. Both checks are synchronous Java calls — they do not add a network hop.

The `Note over E,W: Workflow monitors for completion` block marks where the workflow sits waiting, polling the entity every 30 seconds.

## State machine

Eight meaningful states. The interesting branches:

- From CREATED, a successful assignment produces ASSIGNED; a blocked assignment keeps the task in CREATED (the block is logged but the task is re-eligible for the next `PlannerWorkflow` run).
- From NUDGED, each additional nudge increments `nudgeCount` in place (the state stays NUDGED) until `nudgeCount >= 3`, at which point `StaleTaskChecker` escalates.
- COMPLETED can be reached from ASSIGNED, IN_PROGRESS, NUDGED, or ESCALATED — the task always ends cleanly regardless of which path it took.

## Entity model

`PlannerTaskEntity` is the source of truth; it emits ten distinct event types covering the full lifecycle including assignment blocks and eval scoring. `BoardEventQueue` is the upstream audit log — only `BoardSyncConsumer` subscribes to it.

## Defence-in-depth governance flow

Every outbound action that affects a person passes through:
1. **AssignmentAgent or NudgeAgent** — typed AI primitives that produce structured outputs, not raw text.
2. **GuardrailChecker** — synchronous policy check on capacity, content, and authority.
3. **Tool stub** — the actual dispatch (assignTask / sendNudge / escalateTask); replaced by real integrations in production.
4. **PlannerTaskEntity** — event recorded only after the guardrail passes.

Removing any one of these does not silently downgrade the system; each step records its outcome in entity state, so gaps are visible in the audit trail and surfaced by `EvalRunner`.
