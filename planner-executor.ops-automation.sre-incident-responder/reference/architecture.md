# Architecture — sre-incident-responder

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`IncidentEndpoint` is the entry point. An alert submission writes an `AlertSubmitted` event to `AlertQueue` (event-sourced for audit). `AlertRequestConsumer` subscribes and starts an `IncidentWorkflow` per submission. The workflow drives the incident-response loop: it asks `IncidentCommanderAgent` to triage, then on each probe iteration asks it to decide. Probe decisions route to `TelemetryAgent` (for metrics, logs, and traces) or `RunbookAgent` (for procedure lookups). Once the commander proposes a remediation, the workflow pauses at an approval gate backed by `ApprovalEntity` — the SRE decides from the App UI. On approval, `RemediationAgent` executes the action. Every transition writes an event on `IncidentEntity`; `IncidentView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `AlertSimulator` drips sample alerts for demo purposes; `StaleIncidentMonitor` ticks every 60 s to mark long-running `INVESTIGATING` incidents as `TIMED_OUT`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow checks the flag before every probe dispatch. `ApprovalEntity` is keyed by `incidentId` and holds the pending approval and the SRE's decision.

## Interaction sequence

The sequence diagram traces the happy path (J1). The probe loop in the middle is the investigation cycle — each iteration runs check-halt → propose-probe → probe-guardrail → dispatch-probe → record-probe → investigate-decide. The approval sub-sequence shows the workflow pausing in `awaitApprovalStep` and resuming when the SRE submits a decision. The post-remediation section shows the safety evaluator, the report composition step, and the eval scorer running in sequence.

## State machine

`IncidentEntity` has eight states. `TRIAGING` is the initial state. `INVESTIGATING` is the probe loop state — most probe events fire here without changing the status. `AWAITING_APPROVAL` is the pause state for the human approval gate. `REMEDIATING` covers the execution window. A mitigated incident lands in `MITIGATED`; one where the investigation or approval budget was exhausted lands in `UNRESOLVED`; operator or safety halt produces `HALTED`; timeout produces `TIMED_OUT`.

## Entity model

`IncidentEntity` is the system's source of truth; every transition writes one of sixteen event types. `ApprovalEntity` carries the pending action and the SRE's decision. `SystemControlEntity` carries the operator halt flag. `AlertQueue` is the audit log of submitted alerts. `IncidentView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency and timeouts

- Per-step timeouts: `triageStep` 60 s, `proposeProbeStep` 45 s, `dispatchProbeStep` 90 s, `investigateDecideStep` 45 s, `proposeRemediationStep` 60 s, `awaitApprovalStep` 30 minutes, `executeRemediationStep` 120 s, `composeReportStep` 90 s, `scoreIncidentStep` 60 s.
- Replan budget: 2 consecutive `ReplanInvestigation` outputs; the third becomes `FailInvestigation`.
- Failure budget: 3 consecutive `ContinueInvestigation` on the same `(probeKind, target)`; the fourth becomes `FailInvestigation`.
- Approval timeout: 30 minutes; expiry routes to `unresolvedStep`.
- Idempotency: `(description, reportedBy)` over a 30 s window deduplicates `POST /api/incidents`.
- Stale detection: `StaleIncidentMonitor` every 60 s; incidents `INVESTIGATING` for > 10 minutes are marked `TIMED_OUT`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every probe loop iteration; no caching.
