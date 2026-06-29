# Data model — durable-workflow-backed-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MetricSample` | `metricName` | `String` | no | Name of the metric (e.g., `latency_p99_ms`). |
| | `value` | `double` | no | Observed value. |
| | `unit` | `String` | no | Unit of measure (e.g., `ms`, `percent`, `count`). |
| | `sampledAt` | `Instant` | no | When the sample was captured by the DIAGNOSE phase. |
| `LogEntry` | `level` | `String` | no | `ERROR`, `WARN`, or `INFO`. |
| | `message` | `String` | no | Log message text. |
| | `serviceId` | `String` | no | Service that emitted the log line. |
| | `timestamp` | `Instant` | no | When the log line was emitted. |
| `DiagnosisReport` | `rootCause` | `String` | no | Concise root-cause sentence. |
| | `affectedService` | `String` | no | The service identified as the failure origin. |
| | `severity` | `String` | no | `critical`, `high`, or `medium`. |
| | `evidenceMetrics` | `List<MetricSample>` | no | Possibly empty (J6 demonstrates the empty path). |
| | `evidenceLogs` | `List<LogEntry>` | no | Possibly empty. |
| | `diagnosedAt` | `Instant` | no | When the DIAGNOSE task returned. |
| `ActionReceipt` | `receiptId` | `String` | no | Short stable id (`r-<8 hex>`). |
| | `action` | `String` | no | The corrective action applied. |
| | `target` | `String` | no | The service or resource targeted. |
| | `issuedAt` | `Instant` | no | When the action was issued. |
| `ActionStatus` | `receiptId` | `String` | no | Matches the `ActionReceipt.receiptId`. |
| | `state` | `String` | no | `confirmed` or `failed`. |
| | `confirmedAt` | `Instant` | no | When the confirmation was received. |
| `RemediationPlan` | `actionsApplied` | `List<ActionReceipt>` | no | Non-empty on the happy path. |
| | `outcome` | `String` | no | `applied`, `partial`, or `rolled-back`. |
| | `remediatedAt` | `Instant` | no | When the REMEDIATE task returned. |
| `HealthCheck` | `serviceId` | `String` | no | The service checked. |
| | `healthy` | `boolean` | no | `true` when the service responds within normal bounds. |
| | `latencyP99Ms` | `double` | no | P99 latency observed during the check. |
| | `checkedAt` | `Instant` | no | When the health check ran. |
| `ResolutionCheck` | `incidentId` | `String` | no | The incident being confirmed. |
| | `resolved` | `boolean` | no | `true` if the incident is confirmed resolved. |
| | `evidence` | `String` | no | One-sentence description of what confirmed or denied resolution. |
| | `checkedAt` | `Instant` | no | When the resolution check ran. |
| `VerificationResult` | `resolved` | `boolean` | no | Mirrors `resolutionCheck.resolved`. |
| | `healthChecks` | `List<HealthCheck>` | no | All health checks performed in the VERIFY phase. |
| | `resolutionCheck` | `ResolutionCheck` | no | The final resolution confirmation. |
| | `verifiedAt` | `Instant` | no | When the VERIFY task returned. |
| `BudgetViolation` | `phase` | `String` | no | `DIAGNOSE`, `REMEDIATE`, or `VERIFY`. |
| | `tool` | `String` | no | Name of the halted tool. |
| | `callsUsed` | `int` | no | Tool-call count at the moment of halt. |
| | `cap` | `int` | no | The per-phase cap that was reached. |
| | `violatedAt` | `Instant` | no | When `BudgetGuardrail` fired. |
| `IncidentRecord` (entity state) | `incidentId` | `String` | no | — |
| | `alertId` | `Optional<String>` | yes | Populated after `IncidentCreated`. |
| | `serviceId` | `Optional<String>` | yes | Populated after `IncidentCreated`. |
| | `diagnosis` | `Optional<DiagnosisReport>` | yes | Populated after `DiagnosisCompleted`. |
| | `remediation` | `Optional<RemediationPlan>` | yes | Populated after `RemediationCompleted`. |
| | `verification` | `Optional<VerificationResult>` | yes | Populated after `VerificationCompleted`. |
| | `status` | `IncidentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `IncidentCreated` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp (`VERIFIED` or `FAILED`). |
| | `budgetViolations` | `List<BudgetViolation>` | no | Appended on every `BudgetExhausted` event; empty on the happy path. |
| `DurabilityReport` | `reportId` | `String` | no | Short stable id. |
| | `windowStart` | `Instant` | no | Start of the evaluation window. |
| | `windowEnd` | `Instant` | no | End of the evaluation window. |
| | `incidentsEvaluated` | `int` | no | Count of closed incidents in the window. |
| | `completionRatePercent` | `double` | no | Percentage that reached `VERIFIED`. |
| | `budgetViolationRatePercent` | `double` | no | Percentage with ≥ 1 `BudgetExhausted` event. |
| | `mttrMinutes` | `double` | no | Mean time to `VERIFIED` in minutes. |
| | `timeoutRatePercent` | `double` | no | Percentage of `FAILED` incidents with step-timeout cause. |
| | `generatedAt` | `Instant` | no | When `DurabilityEvaluator` produced this report. |

Every nullable lifecycle field on `IncidentRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`IncidentStatus`: `CREATED`, `DIAGNOSING`, `DIAGNOSED`, `REMEDIATING`, `REMEDIATED`, `VERIFYING`, `VERIFIED`, `FAILED`.

`OpsPhase` (used by `@FunctionTool` annotations and `BudgetGuardrail`): `DIAGNOSE`, `REMEDIATE`, `VERIFY`.

## Events (`IncidentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `IncidentCreated` | `alertId: String, serviceId: String` | → CREATED |
| `DiagnoseStarted` | — | → DIAGNOSING |
| `DiagnosisCompleted` | `diagnosis: DiagnosisReport` | → DIAGNOSED |
| `RemediateStarted` | — | → REMEDIATING |
| `RemediationCompleted` | `plan: RemediationPlan` | → REMEDIATED |
| `VerifyStarted` | — | → VERIFYING |
| `VerificationCompleted` | `result: VerificationResult` | → VERIFIED (terminal happy) |
| `BudgetExhausted` | `phase, tool, callsUsed, cap, violatedAt` | no status change (audit-only) |
| `IncidentFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `IncidentRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `budgetViolations = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## Events (`DurabilityReportEntity`)

| Event | Payload | Effect |
|---|---|---|
| `DurabilityReportRecorded` | `report: DurabilityReport` | Replaces latest report; appends to rolling history (capped at 24). |

## View row

`IncidentRow` mirrors `IncidentRecord` exactly. The UI fetches the full row via `GET /api/incidents/{id}` and streams updates via `GET /api/incidents/sse`.

The view declares ONE query: `getAllIncidents: SELECT * AS incidents FROM incident_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`OpsTasks.java`)

```java
public final class OpsTasks {
  public static final Task<DiagnosisReport> DIAGNOSE_INCIDENT = Task
      .name("Diagnose incident")
      .description("Query metrics and fetch logs to identify root cause and severity")
      .resultConformsTo(DiagnosisReport.class);

  public static final Task<RemediationPlan> APPLY_REMEDIATION = Task
      .name("Apply remediation")
      .description("Apply corrective actions and confirm each action's status")
      .resultConformsTo(RemediationPlan.class);

  public static final Task<VerificationResult> VERIFY_FIX = Task
      .name("Verify fix")
      .description("Check service health and confirm the incident is resolved")
      .resultConformsTo(VerificationResult.class);

  private OpsTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Budget-tagged tools

Each `@FunctionTool` method on `DiagnoseTools`, `RemediateTools`, and `VerifyTools` carries an `OpsPhase` constant. `BudgetGuardrail` reads this constant and the per-phase cap before the tool body runs, counts the call, and halts when the cap is reached. The tool registry is built once at startup; the guardrail reads it for every call. Per-phase caps: `DIAGNOSE = 8`, `REMEDIATE = 6`, `VERIFY = 6`.
