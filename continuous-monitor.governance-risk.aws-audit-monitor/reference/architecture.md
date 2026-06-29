# Architecture — aws-audit-monitor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`AuditScanPoller` is the heartbeat — a TimedAction that ticks every 60 s and writes simulated `FindingDetected` events into `FindingQueue` (event-sourced for audit). A `FindingNormalizer` Consumer subscribes to that queue, strips account-identifying ARN segments, maps `ruleId` to a `controlDomain`, and emits `FindingNormalized` on the per-finding `AuditFindingEntity`. That normalized event starts a `FindingWorkflow` instance, which orchestrates: analyze → compile into report → pause for risk-officer review → finalize to PUBLISHED/DISMISSED.

`AccuracyEvalRunner` runs alongside as a second TimedAction, ticking every 4 hours and scoring sampled published findings.

## Interaction sequence

The sequence traces the happy path (J1 + J2 combined). Two distinct pauses occur: (1) the `FindingNormalizer` Consumer's natural subscription lag (sub-second), and (2) the workflow's explicit PENDING_REVIEW pause (open-ended; the workflow polls `AuditReportEntity` every 10 s for a risk-officer decision).

The `Note over RE,U: Workflow pauses` block marks the second pause — this is where the deployer-runtime HITL gate lives, and it is the system's primary safety mechanism. No finding analysis reaches the organization's compliance record without a named reviewer.

## State machine

Seven states. The interesting branches:

- After ANALYZED, the finding is compiled into a report batch. COMPILED transitions to PENDING_REVIEW when the `ReportCompilerAgent` finishes.
- PENDING_REVIEW has two outbound transitions: risk officer Publish → PUBLISHED terminal; risk officer Dismiss → DISMISSED terminal. There is no auto-timeout — reports wait indefinitely. (Deployers wanting an SLA escalation can add a TimedAction that alerts when a report has been in PENDING_REVIEW for longer than N hours.)

## Entity model

`AuditFindingEntity` is the source of truth for each finding; it emits seven distinct event types covering detection through eval. `AuditReportEntity` accumulates finding references and carries the sign-off decision. `FindingQueue` is the upstream audit log — only `FindingNormalizer` subscribes to it.

## Defence-in-depth governance flow

For any finding that reaches PUBLISHED, it has passed through:
1. **FindingNormalizer** — strips account-identifying data before the model sees the finding detail.
2. **FindingAnalystAgent** — typed analyst defaults to HIGH severity under uncertainty.
3. **ReportCompilerAgent** — groups findings by severity; produces an executive summary for human review.
4. **HITL gate** — risk officer must explicitly click Publish; no automated path to PUBLISHED exists.
5. **AccuracyEvalRunner** — periodic scoring surfaces analysis drift before it affects a real compliance cycle.

Each step is independent. Removing any one does not silently lower the bar; it explicitly opens an audit gap that the eval sampler is likely to catch.
