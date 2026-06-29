# Data model — antom-payment

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PaymentInstruction` | `paymentId` | `String` | no | UUID minted by `PaymentEndpoint`. |
| | `type` | `PaymentType` | no | Enum value. |
| | `currency` | `String` | no | ISO 4217 code, e.g. `"USD"`. |
| | `amountMinorUnits` | `long` | no | Amount in minor currency units (cents, fen, etc.). |
| | `recipientId` | `String` | no | Antom merchant or payee identifier. |
| | `method` | `PaymentMethod` | no | Enum value. |
| | `memo` | `String` | no | Optional human note; empty string if not supplied. |
| | `submittedBy` | `String` | no | Operator identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `AntomApiResponse` | `transactionId` | `String` | no | Antom-assigned transaction id. |
| | `antomStatus` | `String` | no | Raw Antom status code, e.g. `"SUCCESS"`, `"PENDING"`. |
| | `settledAmountMinorUnits` | `long` | no | Amount the API confirmed settled. |
| | `feeMinorUnits` | `long` | no | Antom processing fee in minor units. |
| | `currency` | `String` | no | ISO 4217 code from the API response. |
| | `settledAt` | `Instant` | no | API-reported settlement timestamp. |
| `PaymentResult` | `outcome` | `String` | no | `"settled"` / `"queried"` / `"refunded"` / `"error"`. |
| | `apiResponse` | `AntomApiResponse` | no | API response fields. |
| | `agentSummary` | `String` | no | 1–2 sentence summary from the agent. |
| | `decidedAt` | `Instant` | no | When the agent returned the result. |
| `FraudSignal` | `signalType` | `String` | no | e.g. `"velocity-breach"`, `"blocked-recipient"`, `"amount-anomaly"`. |
| | `detail` | `String` | no | Human-readable detail string. |
| | `detectedAt` | `Instant` | no | When the signal was raised. |
| `Payment` (entity state) | `paymentId` | `String` | no | — |
| | `instruction` | `Optional<PaymentInstruction>` | yes | Populated after `PaymentRequested`. |
| | `result` | `Optional<PaymentResult>` | yes | Populated after `PaymentSettled` or `PaymentQueried`. |
| | `fraudSignal` | `Optional<FraudSignal>` | yes | Populated after `FraudSignalDetected`. |
| | `status` | `PaymentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PaymentRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Payment` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`PaymentType`: `INITIATE`, `QUERY`, `REFUND`.
`PaymentMethod`: `CARD`, `BANK_TRANSFER`, `WALLET`.
`PaymentStatus`: `REQUESTED`, `AUTHORIZED`, `REJECTED`, `AWAITING_APPROVAL`, `EXECUTING`, `SETTLED`, `QUERIED`, `HALTED`, `FAILED`.

## Events (`PaymentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PaymentRequested` | `instruction` | → REQUESTED |
| `PaymentAuthorized` | — | → AUTHORIZED |
| `PaymentRejected` | `reason: String` | → REJECTED (terminal) |
| `ApprovalRequired` | — | → AWAITING_APPROVAL |
| `ApprovalGranted` | `approvedBy: String` | → AWAITING_APPROVAL (workflow resumes to EXECUTING) |
| `ApprovalDenied` | `deniedBy: String` | → REJECTED (terminal) |
| `ExecutionStarted` | — | → EXECUTING |
| `PaymentSettled` | `result` | → SETTLED (terminal happy — INITIATE / REFUND) |
| `PaymentQueried` | `result` | → QUERIED (terminal happy — QUERY) |
| `FraudSignalDetected` | `fraudSignal` | (triggers halt) |
| `PaymentHalted` | `signalType: String` | → HALTED (terminal) |
| `PaymentFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Payment.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PaymentRow` mirrors `Payment`. The UI displays all fields including `instruction`, `result`, and `fraudSignal`. No fields are stripped from the view row — unlike the docreview blueprint, payment data does not contain a raw/sanitized split.

The view declares ONE query: `getAllPayments: SELECT * AS payments FROM payment_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`PaymentTasks.java`)

```java
public final class PaymentTasks {
  public static final Task<PaymentResult> EXECUTE_PAYMENT = Task
      .name("Execute payment")
      .description("Invoke the appropriate Antom API for the authorized payment instruction and return a PaymentResult")
      .resultConformsTo(PaymentResult.class);

  private PaymentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
