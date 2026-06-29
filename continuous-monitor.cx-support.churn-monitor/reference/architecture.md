# Architecture — churn-monitor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`AccountPoller` is the heartbeat — a TimedAction that ticks every 60 s and writes simulated `SnapshotReceived` events into `AccountSnapshotQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts personally identifiable fields, and emits `SnapshotSanitized` on the per-account `AccountChurnEntity`. That sanitized event starts a `ChurnWorkflow` instance, which orchestrates: score → (if HIGH risk) advise → either wait for a manager action or auto-close for MEDIUM and LOW risk.

`DriftFairnessEvalRunner` runs alongside as a second TimedAction, ticking every 6 hours and evaluating batches of scored accounts for prediction drift and demographic bias.

## Interaction sequence

The sequence traces the happy path for a HIGH-risk account (J1 + J2 combined). The key asymmetry compared to a pure approval workflow: LOW and MEDIUM accounts never reach AWAITING_ACTION — they auto-close inside the workflow immediately after scoring. Only HIGH-risk accounts pause, waiting for a customer-success manager to act.

## State machine

Nine states. The branching at SCORED is the critical design point:

- `riskLevel = LOW` → immediate `LOW_RISK_CLOSED` terminal. No human attention consumed.
- `riskLevel = MEDIUM` → immediate `MEDIUM_RISK_CLOSED` terminal. No human attention consumed.
- `riskLevel = HIGH` → `ADVISED` → `AWAITING_ACTION` → manager decides `ACTIONED` or `DISMISSED`.

There is no auto-timeout on `AWAITING_ACTION` — HIGH risk accounts wait indefinitely. Deployers wanting a staleness SLA can add a TimedAction that escalates accounts not actioned after N days.

## Entity model

`AccountChurnEntity` is the source of truth for a single account's scoring lifecycle; it emits eight event types covering the full lifecycle including eval. `AccountSnapshotQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it, preserving the pre-sanitization record for audit purposes.

## Governance architecture

Every account that receives a HIGH risk score and subsequently gets actioned has passed through:

1. **PII sanitizer** — the model scored behavioural signals, not customer identifiers.
2. **ChurnScoringAgent** — typed scorer defaults to HIGH under uncertainty, erring on the side of caution.
3. **RetentionAdvisorAgent** — produces a ranked action plan grounded in the deployer's offered playbook; cannot invent offers.
4. **Manager review gate** — AWAITING_ACTION requires explicit human acknowledgement for HIGH-risk accounts.
5. **Drift + fairness eval** — periodic batch evaluation surfaces score distribution shifts and inter-group disparities before they affect business decisions at scale.

Each mechanism is independent. Removing the PII sanitizer silently increases the model's exposure to identifiers without changing scores — the drift eval is unlikely to catch that. Removing the fairness eval allows demographic bias to accumulate invisibly. The design makes all five components explicit so deployers understand what they are keeping and what they are trading away.
