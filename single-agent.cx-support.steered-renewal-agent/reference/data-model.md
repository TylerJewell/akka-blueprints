# Data model — steered-renewal-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PatronRecord` | `patronId` | `String` | no | Stable patron identifier. |
| | `displayName` | `String` | no | Human-readable name for the UI. |
| | `patronTier` | `String` | no | `STANDARD`, `PREMIUM`, or `STAFF`. |
| | `outstandingFinesCents` | `double` | no | Fine balance in cents; drives the guardrail's fine-threshold check. |
| | `lifetimeRenewals` | `int` | no | Total renewals across all loans, all time. |
| `LoanDetail` | `loanId` | `String` | no | Stable loan identifier. |
| | `itemBarcode` | `String` | no | Physical item barcode. |
| | `itemTitle` | `String` | no | Human-readable title for the UI. |
| | `itemType` | `String` | no | `BOOK`, `PERIODICAL`, or `MEDIA`. |
| | `originalDueDate` | `Instant` | no | When the item was first due. |
| | `priorRenewalCount` | `int` | no | Number of renewals already granted for this loan. |
| | `maxRenewalsAllowed` | `int` | no | Policy cap for this item type. |
| `RenewalRequest` | `renewalId` | `String` | no | UUID minted by `RenewalEndpoint`. |
| | `patronId` | `String` | no | The requesting patron. |
| | `loanId` | `String` | no | The loan being renewed. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `RenewalDecision` | `outcome` | `Outcome` | no | Enum value. |
| | `reason` | `String` | no | One sentence. |
| | `newDueDate` | `Optional<Instant>` | yes | Present for APPROVED and EXTENDED; absent for DENIED. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `NotificationRecord` | `channel` | `String` | no | `in-app`, `email`, or `sms`. |
| | `message` | `String` | no | Human-readable notification text. |
| | `sentAt` | `Instant` | no | When `notifyStep` completed. |
| `Renewal` (entity state) | `renewalId` | `String` | no | — |
| | `request` | `Optional<RenewalRequest>` | yes | Populated after `RenewalRequested`. |
| | `patron` | `Optional<PatronRecord>` | yes | Populated after `LoanEnriched`. |
| | `loan` | `Optional<LoanDetail>` | yes | Populated after `LoanEnriched`. |
| | `decision` | `Optional<RenewalDecision>` | yes | Populated after `DecisionRecorded`. |
| | `notification` | `Optional<NotificationRecord>` | yes | Populated after `NotificationSent`. |
| | `status` | `RenewalStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RenewalRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Renewal` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Outcome`: `APPROVED`, `EXTENDED`, `DENIED`.
`RenewalStatus`: `REQUESTED`, `ENRICHED`, `DECIDING`, `DECISION_RECORDED`, `COMPLETED`, `FAILED`.

## Events (`LoanEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RenewalRequested` | `request` | → REQUESTED |
| `LoanEnriched` | `patron`, `loan` | → ENRICHED |
| `DecisionStarted` | — | → DECIDING |
| `DecisionRecorded` | `decision` | → DECISION_RECORDED |
| `NotificationSent` | `notification` | → COMPLETED (terminal happy) |
| `RenewalFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Renewal.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RenewalRow` mirrors `Renewal` exactly — there is no raw-document equivalent to exclude. The view declares ONE query: `getAllRenewals: SELECT * AS renewals FROM renewal_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`RenewalTasks.java`)

```java
public final class RenewalTasks {
  public static final Task<RenewalDecision> RENEW_LOAN = Task
      .name("Renew loan")
      .description("Read the patron record and loan detail, apply renewal policy, and return a RenewalDecision")
      .resultConformsTo(RenewalDecision.class);

  private RenewalTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Policy constants

The following constants are declared on `PolicyEnforcer` and drive the guardrail checks:

| Constant | Default value | Meaning |
|---|---|---|
| `FINE_THRESHOLD_CENTS` | `1000` | Outstanding fines above this amount block APPROVED and EXTENDED outcomes. |
| `MAX_RENEWAL_WINDOW_DAYS` | `28` | Maximum days in the future a `newDueDate` may be set. |
