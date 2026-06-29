# Data model — sre-incident-responder

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `AlertRequest` | `description` | `String` | no | Free-text alert description. |
| | `severity` | `String` | no | Alert severity: LOW, MEDIUM, HIGH, or CRITICAL. |
| | `reportedBy` | `String` | no | Identifier of the submitting SRE or monitoring system. |
| `InvestigationLedger` | `facts` | `List<String>` | no | Facts the commander believes are established. |
| | `hypotheses` | `List<String>` | no | Current candidate root causes. |
| | `probePlan` | `List<String>` | no | Ordered list of planned probes (3–8). |
| | `currentProbe` | `Optional<ProbeDecision>` | yes | Populated between proposeProbeStep and recordProbeStep; cleared at end-of-iteration. |
| `ProbeDecision` | `probeKind` | `ProbeKind` | no | Which probe type to run. |
| | `target` | `String` | no | Metric name, log stream, trace service, or runbook name. |
| | `rationale` | `String` | no | One-sentence justification. |
| `ProbeResult` | `probeKind` | `ProbeKind` | no | Probe type that ran. |
| | `target` | `String` | no | Echo of the target. |
| | `ok` | `boolean` | no | True if the fixture matched and returned data. |
| | `content` | `String` | no | Raw fixture data formatted as a readable excerpt. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ProbeEntry` | `attempt` | `int` | no | 1-based attempt count for this `(probeKind, target)` pair. |
| | `probeKind` | `ProbeKind` | no | Probe type that ran (or was blocked). |
| | `target` | `String` | no | The probe target. |
| | `verdict` | `ProbeVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / UNSAFE. |
| | `result` | `String` | no | Probe content or blocker message. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `RemediationAction` | `actionKind` | `ActionKind` | no | The type of remediation to apply. |
| | `target` | `String` | no | Service or resource name. |
| | `parameters` | `String` | no | Additional execution parameters (e.g., "rolling-restart", "canary=10%"). |
| | `rationale` | `String` | no | One-sentence justification citing probe evidence. |
| | `estimatedImpact` | `ImpactLevel` | no | LOW / MEDIUM / HIGH / CRITICAL. CRITICAL is blocked by guardrail. |
| `ApprovalRequest` | `incidentId` | `String` | no | The incident this approval is for. |
| | `action` | `RemediationAction` | no | The proposed action awaiting decision. |
| | `requestedAt` | `Instant` | no | When the workflow requested approval. |
| | `sreNote` | `Optional<String>` | yes | Optional context note from the commander. |
| `ApprovalDecision` | `approved` | `boolean` | no | True if the SRE accepted the action. |
| | `decidedBy` | `String` | no | Identifier of the deciding SRE. |
| | `reason` | `String` | no | Acceptance or rejection rationale. |
| | `decidedAt` | `Instant` | no | When the decision was submitted. |
| `ActionOutcome` | `actionKind` | `ActionKind` | no | The action that was executed. |
| | `target` | `String` | no | The service or resource targeted. |
| | `succeeded` | `boolean` | no | True if the control plane reported success. |
| | `observedEffect` | `String` | no | What the control plane reported after execution. |
| | `errorDetail` | `Optional<String>` | yes | Populated when `succeeded=false`. |
| | `executedAt` | `Instant` | no | When execution completed. |
| `RemediationLedger` | `proposedActions` | `List<RemediationAction>` | no | Every action proposed during this incident. |
| | `lastDecision` | `Optional<ApprovalDecision>` | yes | Most recent SRE decision. |
| | `outcomes` | `List<ActionOutcome>` | no | Execution outcomes in order. |
| `PostIncidentReport` | `summary` | `String` | no | 80–140 word executive summary. |
| | `rootCauseDiagnosis` | `String` | no | Specific root cause identified by the commander. |
| | `timeline` | `List<String>` | no | Chronological event list. |
| | `lessonsLearned` | `List<String>` | no | 2–4 concrete improvement bullets. |
| | `followUpActions` | `Optional<String>` | yes | Optional tasks for post-incident follow-up. |
| | `producedAt` | `Instant` | no | When the report was generated. |
| `EvalScore` | `incidentId` | `String` | no | The incident this score covers. |
| | `investigationCoverageScore` | `int` | no | 1–10; did the commander probe all relevant signal types? |
| | `hypothesisAccuracyScore` | `int` | no | 1–10; did the commander's hypotheses match the evidence? |
| | `timeToMitigateMinutes` | `int` | no | Wall-clock minutes from `IncidentCreated` to `IncidentMitigated`. |
| | `findings` | `List<String>` | no | 2–4 brief observations on investigation quality. |
| | `scoredAt` | `Instant` | no | When the eval ran. |
| `Incident` (entity state) | `incidentId` | `String` | no | Unique id. |
| | `description` | `String` | no | Original alert description. |
| | `severity` | `String` | no | Alert severity. |
| | `status` | `IncidentStatus` | no | See enum. |
| | `investigationLedger` | `Optional<InvestigationLedger>` | yes | Populated after `IncidentTriaged`. |
| | `remediationLedger` | `Optional<RemediationLedger>` | yes | Populated after first `RemediationProposed`. |
| | `report` | `Optional<PostIncidentReport>` | yes | Populated after `IncidentMitigated` or `IncidentUnresolved`. |
| | `evalScore` | `Optional<EvalScore>` | yes | Populated after `EvalScoreRecorded`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `IncidentUnresolved` / `IncidentTimedOut`. |
| | `haltReason` | `Optional<String>` | yes | Populated on halt events. |
| | `createdAt` | `Instant` | no | When `IncidentCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the incident reached a terminal state. |
| `ApprovalState` (entity state) | `pendingRequest` | `Optional<ApprovalRequest>` | yes | The current pending approval; empty after a decision is recorded. |
| | `decision` | `Optional<ApprovalDecision>` | yes | The SRE's decision; populated once. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `InvestigationNextStep` | (sealed interface) | — | — | Permits `ContinueInvestigation(ProbeDecision)`, `ReplanInvestigation(InvestigationLedger)`, `ProposeRemediation(RemediationAction)`, `FailInvestigation(String reason)`. |

## Enums

- `ProbeKind` → `METRICS`, `LOGS`, `TRACES`, `RUNBOOK`.
- `ProbeVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `UNSAFE`.
- `ActionKind` → `RESTART_SERVICE`, `ROLLBACK_DEPLOYMENT`, `SHIFT_TRAFFIC`, `SCALE_UP`, `DISABLE_FEATURE_FLAG`.
- `ImpactLevel` → `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.
- `IncidentStatus` → `TRIAGING`, `INVESTIGATING`, `AWAITING_APPROVAL`, `REMEDIATING`, `MITIGATED`, `UNRESOLVED`, `HALTED`, `TIMED_OUT`.

## Events (`IncidentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `IncidentCreated` | `incidentId, description, severity, createdAt` | → TRIAGING |
| `IncidentTriaged` | `investigationLedger` | → INVESTIGATING |
| `ProbeDispatched` | `probeDecision` | no status change; sets `investigationLedger.currentProbe`. |
| `ProbeBlocked` | `attempt, probeDecision, blocker` | no status change; appends a `ProbeEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `ProbeRecorded` | `entry: ProbeEntry` | no status change; appends to investigation ledger probe entries. |
| `InvestigationReplanned` | `revisedLedger: InvestigationLedger` | no status change; replaces `investigationLedger`. |
| `RemediationProposed` | `action: RemediationAction` | → AWAITING_APPROVAL; creates `remediationLedger` if absent. |
| `ApprovalGranted` | `decision: ApprovalDecision` | → REMEDIATING |
| `ApprovalRejected` | `decision: ApprovalDecision` | → INVESTIGATING |
| `RemediationExecuted` | `outcome: ActionOutcome` | no status change if re-investigation follows; appends to `remediationLedger.outcomes`. |
| `IncidentMitigated` | `report: PostIncidentReport` | → MITIGATED, `finishedAt = now` |
| `IncidentUnresolved` | `failureReason` | → UNRESOLVED, `finishedAt = now` |
| `IncidentHaltedAutomatic` | `haltReason` | → HALTED, `finishedAt = now` |
| `IncidentHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `IncidentTimedOut` | `failureReason` | → TIMED_OUT, `finishedAt = now` |
| `EvalScoreRecorded` | `evalScore: EvalScore` | no status change; populates `evalScore`. |

## Events (`ApprovalEntity`)

| Event | Payload |
|---|---|
| `ApprovalRequested` | `incidentId, action, requestedAt` |
| `ApprovalDecided` | `incidentId, decision, decidedAt` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`AlertQueue`)

| Event | Payload |
|---|---|
| `AlertSubmitted` | `incidentId, description, severity, reportedBy, submittedAt` |

## View row

`IncidentRow` mirrors `Incident` minus the heavy probe payload — `investigationLedger` probe entries are truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `result` is capped at 240 characters. The UI fetches the full incident by id on click via `GET /api/incidents/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
