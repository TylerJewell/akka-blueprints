# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## ChangeRequest (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `changeId` | `String` | no | UUID, also the workflow id |
| `targetEnvironment` | `String` | no | Target environment: dev, staging, or production |
| `changeType` | `String` | no | Type of change: deploy, scale, config-update, or rollback |
| `description` | `String` | no | Human-supplied change description |
| `requestedBy` | `String` | no | Identity of the submitter |
| `status` | `ChangeStatus` | no | Lifecycle state |
| `workPlan` | `Optional<WorkPlan>` | yes | Coordinator work plan; null until `WorkPlanEmitted` |
| `infraAssessment` | `Optional<InfraAssessment>` | yes | InfraAgent output; null until `InfraAssessed` |
| `deployAssessment` | `Optional<DeployAssessment>` | yes | DeployAgent output; null until `DeployAssessed` |
| `obsAssessment` | `Optional<ObsAssessment>` | yes | ObservabilityAgent output; null until `ObsAssessed` |
| `report` | `Optional<ReadinessReport>` | yes | Consolidated report; null until `ReportConsolidated` |
| `failureReason` | `Optional<String>` | yes | Set on `ChangeBlocked`, `ChangeDegraded`, or `ChangeHalted` |
| `approvedBy` | `Optional<String>` | yes | Set on `ChangeApproved` |
| `createdAt` | `Instant` | no | Change creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `OpsView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record ChangeSubmission(String targetEnvironment, String changeType,
                        String description, String requestedBy) {}

record WorkPlan(String infraQuery, String deployQuery,
                String obsQuery, String targetEnvironment) {}

record InfraAssessment(String driftSummary, List<String> quotaWarnings,
                       List<String> dependencyIssues, RiskLevel riskLevel,
                       Instant assessedAt) {}

record DeployAssessment(String manifestSummary, boolean rollbackAvailable,
                        String canaryStatus, RiskLevel riskLevel,
                        Instant assessedAt) {}

record ObsAssessment(List<String> firingAlerts, double sloBurnRate,
                     boolean onCallCovered, RiskLevel riskLevel,
                     Instant assessedAt) {}

record ReadinessReport(String summary, InfraAssessment infraAssessment,
                       DeployAssessment deployAssessment, ObsAssessment obsAssessment,
                       RiskLevel riskLevel, String guardrailVerdict,
                       Instant consolidatedAt) {}
```

## Status enum

```java
enum ChangeStatus {
    PLANNING, IN_PROGRESS, AWAITING_APPROVAL,
    APPROVED, REJECTED, DEGRADED, BLOCKED, HALTED
}
```

## Risk level enum

```java
enum RiskLevel { LOW, MEDIUM, HIGH, CRITICAL }
```

## Events

### ChangeRequestEntity

| Event | Trigger |
|---|---|
| `ChangeCreated` | Workflow creates the change request (`createChange`) |
| `WorkPlanEmitted` | Coordinator returns a `WorkPlan` |
| `InfraAssessed` | InfraAgent returns an `InfraAssessment` |
| `DeployAssessed` | DeployAgent returns a `DeployAssessment` |
| `ObsAssessed` | ObservabilityAgent returns an `ObsAssessment` |
| `ReportConsolidated` | Coordinator returns a `ReadinessReport` |
| `ApprovalRequested` | Workflow parks for a production target (`requestApproval`) |
| `ChangeApproved` | Operator approves via `POST /api/ops/changes/{id}/approve` |
| `ChangeRejected` | Operator rejects via `POST /api/ops/changes/{id}/reject` |
| `ChangeDegraded` | A specialist timed out; report consolidated from partial input |
| `ChangeBlocked` | Guardrail intercepted a destructive tool call |
| `ChangeHalted` | Operator halt signal received via `HaltMonitor` |

### PipelineQueue

| Event | Trigger |
|---|---|
| `ChangeSubmitted` | `submitChange(targetEnvironment, changeType, description, requestedBy)` from endpoint or simulator |

Fields: `{ changeId, targetEnvironment, changeType, description, requestedBy, submittedAt }`.

### HaltSignalEntity

| Event | Trigger |
|---|---|
| `HaltIssued` | `issueHalt(reason)` from `POST /api/ops/halt` |
| `HaltCleared` | `clearHalt(signalId)` from operator |

Fields for `HaltIssued`: `{ signalId, reason, issuedAt }`. Fields for `HaltCleared`: `{ signalId }`.
