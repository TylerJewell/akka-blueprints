# Architecture — live-eval-harness

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`DecisionPoller` is the heartbeat — a TimedAction ticking every 10 s that writes simulated `DecisionReceived` events into `DecisionQueue` (event-sourced for the intake audit log). A `DecisionSanitizer` Consumer subscribes to that queue, strips user-identifiable fields, and emits `DecisionSanitized` on the per-decision `DecisionEntity`. That sanitized event starts an `EvalOrchestrationWorkflow` instance, which calls `RubricEvalAgent` and stores the score, then checks the score against the alarm threshold and emits either `DecisionOk` or `EvalAlarm`.

`DriftSampler` runs alongside as a second TimedAction, ticking every 15 minutes. It reads the recent scored window from `EvalView`, calls `DriftWatchAgent`, and records the resulting `DriftAssessment` on the singleton `DriftSnapshotEntity`.

## Interaction sequence

The sequence traces the alarmed path (J1 + J2 combined): a decision arrives, is sanitized, scores below threshold, and an `EvalAlarm` event propagates through the view to the SSE stream. The second pause — `DriftSampler`'s 15-minute interval — is not shown inline; it is an independent cadence.

Note: there is no human approval gate in this system. The harness is monitoring-only. Human governance officers respond to alarms out-of-band; the harness records the fact of the alarm, not a resolution.

## State machine

Five states. The interesting branch:

- After SANITIZED the eval always runs (with a timeout backstop). EVALUATED has two outbound transitions: `DecisionOk` if score > threshold → OK terminal; `EvalAlarm` if score <= threshold → ALARMED terminal. Both are terminal — once a decision is scored, its state is immutable.
- There is no retry. A timed-out eval emits a sentinel score of 1 (which will trigger ALARMED) rather than leaving the decision in SANITIZED indefinitely.

## Entity model

`DecisionEntity` is the source of truth for each scored decision; it emits five event types. `DriftSnapshotEntity` holds the singleton drift state and emits three event types. `DecisionQueue` is the intake audit log — only `DecisionSanitizer` subscribes to it. `EvalView` joins both `DecisionEntity` and `DriftSnapshotEntity` to give the UI a single query surface.

## Governance data flow

Every decision that enters the system passes through:
1. **DecisionSanitizer** — user-identifiable fields never reach the LLM.
2. **RubricEvalAgent** — scored on arrival; the score is immutable.
3. **Alarm check** — any decision below threshold immediately surfaces an alert.
4. **DriftWatchAgent** — every 15 minutes, aggregate patterns are reviewed for distributional shift.

Removing step 2 silences the per-decision evidence trail. Removing step 4 leaves gradual drift invisible until an individual alarm count crosses a manual review threshold. Both are needed for production-grade coverage.
