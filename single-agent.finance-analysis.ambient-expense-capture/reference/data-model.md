# Data model — ambient-expense-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ExpenseCategory` | `categoryId` | `String` | no | Stable id from the taxonomy (e.g., `meals-and-entertainment`). |
| | `displayName` | `String` | no | Human-readable label for the UI. |
| | `glCode` | `String` | no | General-ledger code for ERP posting. |
| | `maxAmountUsd` | `BigDecimal` | no | Per-line-item policy limit in USD. |
| `SubmitReceiptRequest` | `expenseId` | `String` | no | UUID minted by `ExpenseEndpoint`. |
| | `rawReceiptText` | `String` | no | Pre-sanitization receipt or transcript body. Audit-only. |
| | `submittedBy` | `String` | no | Employee identifier. |
| | `tripCode` | `Optional<String>` | yes | Optional project or trip reference. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedReceipt` | `redactedText` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["payment-card-number","person-name","address"]`. |
| `ExpenseLineItem` | `lineItemId` | `String` | no | Stable id minted by the agent (e.g., `item-1`). |
| | `description` | `String` | no | Short description from the receipt or agent inference. |
| | `amount` | `BigDecimal` | no | Extracted amount. |
| | `currencyCode` | `String` | no | ISO 4217 three-letter code. |
| | `categoryId` | `String` | no | MUST equal a `categoryId` in the taxonomy. |
| | `merchantName` | `String` | no | Merchant name; `"Unknown"` if not determinable. |
| | `expenseDate` | `LocalDate` | no | ISO-8601 date of the charge. |
| | `lineStatus` | `LineItemStatus` | no | Enum value. |
| | `blockReason` | `Optional<String>` | yes | Guardrail rejection message; null when `lineStatus != BLOCKED`. |
| `ExpenseReport` | `expenseId` | `String` | no | Matches the submission id. |
| | `lineItems` | `List<ExpenseLineItem>` | no | One entry per extracted charge. |
| | `totalAmount` | `BigDecimal` | no | Sum of all `amount` values regardless of status. |
| | `currencyCode` | `String` | no | Dominant currency for the report. |
| | `reportStatus` | `ReportStatus` | no | Enum value. |
| | `capturedAt` | `Instant` | no | When the agent returned. |
| `ExpenseSubmission` (entity state) | `expenseId` | `String` | no | — |
| | `request` | `Optional<SubmitReceiptRequest>` | yes | Populated after `ReceiptSubmitted`. |
| | `sanitized` | `Optional<SanitizedReceipt>` | yes | Populated after `ReceiptSanitized`. |
| | `report` | `Optional<ExpenseReport>` | yes | Populated after `ReportReady`. |
| | `status` | `SubmissionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ReceiptSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ExpenseSubmission` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`LineItemStatus`: `APPROVED`, `FLAGGED`, `BLOCKED`.
`ReportStatus`: `FULLY_APPROVED`, `PARTIALLY_BLOCKED`, `FULLY_BLOCKED`.
`SubmissionStatus`: `SUBMITTED`, `SANITIZED`, `CAPTURING`, `REPORT_READY`, `SUBMITTED_TO_SYSTEM`, `FAILED`.

## Events (`ExpenseEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ReceiptSubmitted` | `request` | → SUBMITTED |
| `ReceiptSanitized` | `sanitized` | → SANITIZED |
| `CaptureStarted` | — | → CAPTURING |
| `ReportReady` | `report` | → REPORT_READY |
| `SystemSubmitted` | — | → SUBMITTED_TO_SYSTEM (terminal happy) |
| `SubmissionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ExpenseSubmission.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ExpenseRow` mirrors `ExpenseSubmission` minus `request.rawReceiptText` (the audit log keeps that). The UI fetches the raw receipt on demand via `GET /api/expenses/{id}` and reads `request.rawReceiptText` from the JSON.

The view declares ONE query: `getAllSubmissions: SELECT * AS submissions FROM expense_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ExpenseTasks.java`)

```java
public final class ExpenseTasks {
  public static final Task<ExpenseReport> CAPTURE_EXPENSE = Task
      .name("Capture expense")
      .description("Read the attached receipt and produce an ExpenseReport with one ExpenseLineItem "
          + "per discernible charge, categorized against the provided taxonomy")
      .resultConformsTo(ExpenseReport.class);

  private ExpenseTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
