# Architecture — cyber-guardian-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`TelemetryPoller` is the heartbeat — a TimedAction that ticks every 10 s and writes simulated `SignalReceived` events into `TelemetryQueue` (event-sourced for audit). A `ThreatSignalConsumer` Consumer subscribes to that queue, registers each signal in a per-signal `IncidentEntity`, and starts an `IncidentWorkflow` instance. The workflow orchestrates: classify → branch on severity → (CRITICAL only) halt → (HIGH/CRITICAL) remediation playbook → close.

`EvalEventEmitter` runs alongside as a second TimedAction, ticking every 5 minutes and emitting structured `EvalEventPayload` records for every incident that has closed without one.

## Interaction sequence

The sequence traces the CRITICAL-severity path (J1). Two distinct agent calls happen before a human is involved: `ThreatClassifierAgent` determines severity, then `SafetyHaltAgent` issues the isolation directive without waiting for a human. The `Note over E,EE: EvalEventEmitter tick` block marks the asynchronous eval emission — it fires on the next scheduler tick after the incident closes, not inline.

The human's role is narrowed: the operator can lift a halt (transitioning HALTED → REMEDIATION) and review the playbook, but isolation itself is automatic.

## State machine

Six states. The branching logic:

- After CLASSIFIED, severity determines the path: CRITICAL → HALTED (then operator must lift to REMEDIATION); HIGH → REMEDIATION directly; MEDIUM → CLOSED immediately; LOW → IGNORED (terminal, treated like CLOSED for eval purposes).
- HALTED is not terminal — the operator can always lift it. There is no auto-expiry on HALTED status in the blueprint (deployers may add a `StaleHaltEscalator` TimedAction to alert if a halt is not lifted within N hours).
- Both CLOSED and IGNORED are terminal states that trigger the eval event emission path.

## Entity model

`IncidentEntity` is the source of truth, emitting seven distinct event types covering classification, halt lifecycle, playbook generation, closure, and eval recording. `TelemetryQueue` is the upstream audit log — only `ThreatSignalConsumer` subscribes to it. The raw `rawPayload` field is retained in the queue's audit log but stripped from the `IncidentRow` projected by `IncidentView` (the UI fetches detail on demand via `GET /api/threats/{id}`).

## Governance flow

For any CRITICAL incident, the path is:

1. **ThreatClassifierAgent** — typed assessment; defaults to HIGH under uncertainty.
2. **SafetyHaltAgent** — isolation directive issued automatically; no approval required; no self-lift possible.
3. **RemediationAdvisorAgent** — playbook produced for human review.
4. **Operator lift** — human explicitly lifts the halt to proceed to REMEDIATION.
5. **EvalEventEmitter** — structured eval record emitted on close for downstream audit.

Each step is independent. The halt mechanism does not depend on the eval mechanism, and vice versa. Removing either mechanism does not silently compromise the other.
