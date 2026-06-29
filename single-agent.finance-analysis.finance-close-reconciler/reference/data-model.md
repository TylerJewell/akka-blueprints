# Data model — finance-close-reconciler

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `AccountRule` | `ruleId` | `String` | no | Stable id supplied by the accountant. |
| | `accountPair` | `String` | no | Two account codes separated by `/`, e.g. `"1000/2000"`. |
| | `description` | `String` | no | What the rule checks. |
| | `toleranceAmount` | `String` | no | Absolute amount or percentage string, e.g. `"0.01"` or `"0.5%"`. |
| | `material` | `boolean` | no | Whether a variance on this rule triggers `HAS_VARIANCES` on the report. |
| `TrialBalanceSubmission` | `periodId` | `String` | no | UUID minted by `ReconciliationEndpoint`. |
| | `periodLabel` | `String` | no | User-supplied label, e.g. `"Q2 FY2026"`. |
| | `rawBalance` | `String` | no | Pre-masking CSV body. Audit-only. |
| | `rules` | `List<AccountRule>` | no | Submitted list (1–N). |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedBalance` | `maskedBalance` | `String` | no | Confidential fields masked; this is what the agent sees. |
| | `maskedFieldTypes` | `List<String>` | no | e.g. `["margin-pct","entity-code","internal-code"]`. |
| `AccountFinding` | `ruleId` | `String` | no | MUST equal a submitted `ruleId`. |
| | `accountPair` | `String` | no | Account pair from the rule. |
| | `finding` | `BalanceFinding` | no | Enum value. |
| | `expectedBalance` | `BigDecimal` | no | Balance expected per the rule. |
| | `actualBalance` | `BigDecimal` | no | Balance read from the masked CSV. |
| | `variance` | `BigDecimal` | no | `actualBalance - expectedBalance`; 0.00 if IN_BALANCE. |
| | `note` | `String` | no | Concise explanation or `"Within tolerance."` |
| `ReconciliationReport` | `status` | `ReportStatus` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `findings` | `List<AccountFinding>` | no | One entry per submitted rule. |
| | `proposedGlWrites` | `List<String>` | no | Tool-call outputs that passed the guardrail. |
| | `reconciledAt` | `Instant` | no | When the agent returned. |
| `SignOff` | `approvedBy` | `String` | no | Identity of the accountant who signed off. |
| | `decision` | `SignOffDecision` | no | Enum value. |
| | `note` | `String` | no | Optional explanation (empty string if none). |
| | `decidedAt` | `Instant` | no | When the sign-off endpoint was called. |
| `AttestationResult` | `complete` | `boolean` | no | Whether all required events and findings are present. |
| | `attestedBy` | `String` | no | Copied from `signOff.approvedBy`. |
| | `attestedAt` | `Instant` | no | When `AttestationScorer` ran. |
| `Period` (entity state) | `periodId` | `String` | no | — |
| | `submission` | `Optional<TrialBalanceSubmission>` | yes | Populated after `PeriodSubmitted`. |
| | `sanitized` | `Optional<SanitizedBalance>` | yes | Populated after `BalanceSanitized`. |
| | `report` | `Optional<ReconciliationReport>` | yes | Populated after `ReportRecorded`. |
| | `signOff` | `Optional<SignOff>` | yes | Populated after `SignOffGranted` or `SignOffRejected`. |
| | `attestation` | `Optional<AttestationResult>` | yes | Populated after `AttestationCompleted`. |
| | `status` | `PeriodStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PeriodSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Period` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BalanceFinding`: `IN_BALANCE`, `VARIANCE`, `UNMATCHED`.
`ReportStatus`: `IN_BALANCE`, `HAS_VARIANCES`, `INCOMPLETE`.
`SignOffDecision`: `APPROVED`, `REJECTED`.
`PeriodStatus`: `SUBMITTED`, `SANITIZED`, `RECONCILING`, `REPORT_RECORDED`, `AWAITING_SIGNOFF`, `ATTESTED`, `REJECTED`, `FAILED`.

## Events (`PeriodEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PeriodSubmitted` | `submission` | → SUBMITTED |
| `BalanceSanitized` | `sanitized` | → SANITIZED |
| `ReconciliationStarted` | — | → RECONCILING |
| `ReportRecorded` | `report` | → REPORT_RECORDED |
| `SignOffRequested` | — | → AWAITING_SIGNOFF |
| `SignOffGranted` | `signOff` | (in AWAITING_SIGNOFF; workflow advances to attestStep) |
| `SignOffRejected` | `signOff` | → REJECTED (terminal) |
| `AttestationCompleted` | `attestation` | → ATTESTED (terminal happy) |
| `PeriodFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Period.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PeriodRow` mirrors `Period` minus `submission.rawBalance` (the audit log keeps that). The UI fetches the raw balance on demand via `GET /api/periods/{id}` and reads `submission.rawBalance` from the JSON.

The view declares ONE query: `getAllPeriods: SELECT * AS periods FROM period_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ReconciliationTasks.java`)

```java
public final class ReconciliationTasks {
  public static final Task<ReconciliationReport> RECONCILE_PERIOD = Task
      .name("Reconcile period")
      .description("Read the attached sanitized trial balance and produce a ReconciliationReport with one AccountFinding per submitted rule")
      .resultConformsTo(ReconciliationReport.class);

  private ReconciliationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
