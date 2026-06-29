# Data model — month-end-closer

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `LedgerLine` | `accountCode` | `String` | no | Account code from the chart of accounts. |
| | `description` | `String` | no | Posting description. |
| | `debitAmount` | `BigDecimal` | no | Non-negative debit amount (0 if credit-only line). |
| | `creditAmount` | `BigDecimal` | no | Non-negative credit amount (0 if debit-only line). |
| | `postingDate` | `LocalDate` | no | Date the line was posted to the ledger. |
| `AccountEntry` | `code` | `String` | no | Unique account code. |
| | `name` | `String` | no | Account name. |
| | `type` | `String` | no | `ASSET`, `LIABILITY`, `EQUITY`, `REVENUE`, or `EXPENSE`. |
| `ChartOfAccounts` | `accounts` | `List<AccountEntry>` | no | All valid account codes for the entity. |
| `LedgerSnapshot` | `entity` | `String` | no | Entity code (e.g. `ACME-US`). |
| | `period` | `String` | no | Accounting period (e.g. `2026-05`). |
| | `lines` | `List<LedgerLine>` | no | Possibly empty; J5 demonstrates the empty path. |
| | `chartOfAccounts` | `ChartOfAccounts` | no | Fetched alongside the ledger lines. |
| | `gatheredAt` | `Instant` | no | When the GATHER task returned. |
| `JournalEntry` | `entryId` | `String` | no | Short stable id (`je-<3 digits>`). |
| | `debitAccount` | `String` | no | MUST equal an `AccountEntry.code` from the chart of accounts. |
| | `creditAccount` | `String` | no | MUST equal an `AccountEntry.code` from the chart of accounts. |
| | `amount` | `BigDecimal` | no | Positive entry amount. |
| | `narration` | `String` | no | Entry description. If contains "accrual", `reversalDate` must be non-empty. |
| | `reversalDate` | `Optional<LocalDate>` | yes | Required for accrual entries (E1 rule 3). |
| `JournalEntrySet` | `entries` | `List<JournalEntry>` | no | Possibly empty. |
| | `validatedAt` | `Instant` | no | When the VALIDATE task returned. |
| `TrialBalanceSummary` | `totalDebits` | `BigDecimal` | no | Sum of all entry amounts on the debit side. |
| | `totalCredits` | `BigDecimal` | no | Sum of all entry amounts on the credit side. |
| | `variance` | `BigDecimal` | no | `totalDebits - totalCredits`. |
| | `balanced` | `boolean` | no | `true` iff `variance == 0` (E1 rule 1). |
| `ClosePackage` | `title` | `String` | no | 1-line close report title. |
| | `period` | `String` | no | Accounting period. |
| | `entity` | `String` | no | Entity code. |
| | `trialBalance` | `TrialBalanceSummary` | no | Trial balance for the full entry set. |
| | `journalEntries` | `List<JournalEntry>` | no | All entries included in the close. |
| | `varianceCommentary` | `String` | no | One paragraph explaining any variances. |
| | `writtenAt` | `Instant` | no | When the REPORT task returned. |
| `ReconciliationResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `ReconciliationScorer` finished. |
| `ApprovalRecord` | `step` | `String` | no | `GATHER` or `VALIDATE`. |
| | `approver` | `String` | no | Approver identity (email or username). |
| | `decision` | `String` | no | `APPROVED` or `REJECTED`. |
| | `comment` | `Optional<String>` | yes | Free-text reviewer comment. |
| | `decidedAt` | `Instant` | no | When the approval or rejection was recorded. |
| `CloseRunRecord` (entity state) | `closeRunId` | `String` | no | — |
| | `entity` | `Optional<String>` | yes | Populated after `CloseRunCreated`. |
| | `period` | `Optional<String>` | yes | Populated after `CloseRunCreated`. |
| | `ledgerSnapshot` | `Optional<LedgerSnapshot>` | yes | Populated after `LedgerDataGathered`. |
| | `journalEntrySet` | `Optional<JournalEntrySet>` | yes | Populated after `EntriesValidated`. |
| | `closePackage` | `Optional<ClosePackage>` | yes | Populated after `CloseReportWritten`. |
| | `reconciliation` | `Optional<ReconciliationResult>` | yes | Populated after `ReconciliationScored`. |
| | `approvals` | `List<ApprovalRecord>` | no | Appended on every `StepApproved` and `StepRejected` event; empty at creation. |
| | `status` | `CloseRunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CloseRunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `CloseRunRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`CloseRunStatus`: `CREATED`, `GATHERING`, `AWAITING_GATHER_APPROVAL`, `VALIDATING`, `AWAITING_VALIDATE_APPROVAL`, `REPORTING`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations): `GATHER`, `VALIDATE`, `REPORT`.

## Events (`CloseRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CloseRunCreated` | `entity: String, period: String` | → CREATED |
| `GatherStarted` | — | → GATHERING |
| `LedgerDataGathered` | `ledgerSnapshot: LedgerSnapshot` | → AWAITING_GATHER_APPROVAL |
| `StepApproved` | `step, approver, comment, decidedAt` | AWAITING_GATHER_APPROVAL → VALIDATING; AWAITING_VALIDATE_APPROVAL → REPORTING |
| `StepRejected` | `step, approver, comment, decidedAt` | AWAITING_GATHER_APPROVAL → GATHERING; AWAITING_VALIDATE_APPROVAL → VALIDATING |
| `ValidateStarted` | — | → VALIDATING |
| `EntriesValidated` | `journalEntrySet: JournalEntrySet` | → AWAITING_VALIDATE_APPROVAL |
| `ReportStarted` | — | → REPORTING |
| `CloseReportWritten` | `closePackage: ClosePackage` | (no status change — evalStep follows immediately) |
| `ReconciliationScored` | `reconciliation: ReconciliationResult` | → EVALUATED (terminal happy) |
| `CloseRunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `CloseRunRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `approvals = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`CloseRunRow` mirrors `CloseRunRecord` exactly. The UI fetches the full row via `GET /api/close-runs/{id}` and streams updates via `GET /api/close-runs/sse`.

The view declares ONE query: `getAllCloseRuns: SELECT * AS closeRuns FROM close_run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`CloseTasks.java`)

```java
public final class CloseTasks {
  public static final Task<LedgerSnapshot> GATHER_LEDGER_DATA = Task
      .name("Gather ledger data")
      .description("Fetch ledger lines and chart of accounts for the given entity and period")
      .resultConformsTo(LedgerSnapshot.class);

  public static final Task<JournalEntrySet> VALIDATE_ENTRIES = Task
      .name("Validate entries")
      .description("Check debit/credit balance and validate account codes for the proposed journal entry set")
      .resultConformsTo(JournalEntrySet.class);

  public static final Task<ClosePackage> WRITE_CLOSE_REPORT = Task
      .name("Write close report")
      .description("Compose a ClosePackage with trial balance summary, journal entry list, and variance commentary")
      .resultConformsTo(ClosePackage.class);

  private CloseTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `GatherTools`, `ValidateTools`, and `ReportTools` carries a `Phase` constant. The phase constant is used by the workflow to construct the `TaskDef.metadata("phase", ...)` tag passed to each task. The agent system prompt instructs the model to call only tools matching the current task's phase; the deterministic `ReconciliationScorer` cross-checks the result for account-code validity without relying on the agent's phase discipline.
