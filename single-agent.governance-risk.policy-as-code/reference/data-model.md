# Data model — policy-as-code

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PolicyRule` | `ruleId` | `String` | no | Stable id supplied by the user. |
| | `text` | `String` | no | What the rule prohibits or requires. |
| | `category` | `String` | no | e.g. `network-access`, `encryption`, `version-pinning`. |
| | `severityFloor` | `String` | no | Lowest severity a violation may carry (`LOW`/`MEDIUM`/`HIGH`/`CRITICAL`). |
| `ChangeRequest` | `changeId` | `String` | no | UUID minted by `PolicyEndpoint`. |
| | `changeTitle` | `String` | no | User-supplied label. |
| | `changeType` | `String` | no | `terraform-plan`, `k8s-manifest`, `dependency-bump`, or `custom`. |
| | `rawPayload` | `String` | no | Pre-normalization change body. Audit-only. |
| | `rules` | `List<PolicyRule>` | no | Submitted policy rules (1–N). |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ValidatedChange` | `changeType` | `String` | no | Detected or confirmed change type. |
| | `affectedResources` | `List<String>` | no | e.g. `["aws_s3_bucket.audit-logs"]`. |
| | `normalizedPayload` | `String` | no | Comment-stripped, whitespace-collapsed form. This is what the agent sees. |
| `Violation` | `ruleId` | `String` | no | MUST equal a submitted `ruleId`. |
| | `severity` | `Severity` | no | Enum value. |
| | `affectedResource` | `String` | no | e.g. `"aws_s3_bucket_public_access_block.audit-logs"`. |
| | `evidence` | `String` | no | Short passage or field value from the change triggering this rule. |
| | `recommendation` | `String` | no | Actionable verb-phrase. |
| `PolicyDecision` | `outcome` | `Outcome` | no | Enum value. |
| | `rationale` | `String` | no | 1–3 sentences. |
| | `violations` | `List<Violation>` | no | One entry per breached rule; empty on ALLOW. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `GateResult` | `status` | `GateStatus` | no | `OPEN` or `BLOCKED`. |
| | `reason` | `String` | no | One sentence explaining the gate decision. |
| | `evaluatedAt` | `Instant` | no | When `GateEvaluator` finished. |
| `ChangeEvaluation` (entity state) | `changeId` | `String` | no | — |
| | `request` | `Optional<ChangeRequest>` | yes | Populated after `ChangeSubmitted`. |
| | `validated` | `Optional<ValidatedChange>` | yes | Populated after `ChangeValidated`. |
| | `decision` | `Optional<PolicyDecision>` | yes | Populated after `DecisionRecorded`. |
| | `gate` | `Optional<GateResult>` | yes | Populated after `GateEvaluated`. |
| | `status` | `EvaluationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ChangeSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ChangeEvaluation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Severity`: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.
`Outcome`: `ALLOW`, `WARN`, `DENY`.
`GateStatus`: `OPEN`, `BLOCKED`.
`EvaluationStatus`: `SUBMITTED`, `VALIDATED`, `ENFORCING`, `DECISION_RECORDED`, `GATED`, `FAILED`.

## Events (`ChangeRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ChangeSubmitted` | `request` | → SUBMITTED |
| `ChangeValidated` | `validated` | → VALIDATED |
| `EnforcementStarted` | — | → ENFORCING |
| `DecisionRecorded` | `decision` | → DECISION_RECORDED |
| `GateEvaluated` | `gate` | → GATED (terminal happy) |
| `EvaluationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ChangeEvaluation.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ChangeRow` mirrors `ChangeEvaluation` minus `request.rawPayload` (the audit log keeps that). The UI fetches the raw payload on demand via `GET /api/changes/{id}` and reads `request.rawPayload` from the JSON.

The view declares ONE query: `getAllChanges: SELECT * AS changes FROM policy_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`PolicyTasks.java`)

```java
public final class PolicyTasks {
  public static final Task<PolicyDecision> ENFORCE_POLICY = Task
      .name("Enforce policy")
      .description("Read the attached change payload and evaluate it against every submitted policy rule, returning a PolicyDecision")
      .resultConformsTo(PolicyDecision.class);

  private PolicyTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
