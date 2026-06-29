# Data model — gl-reconciler

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `LedgerEntry` | `entryId` | `String` | no | Unique identifier of the ledger posting. |
| | `accountId` | `String` | no | Chart-of-accounts code. MUST be present in `coa.json`. |
| | `description` | `String` | no | Narrative description of the posting. |
| | `debit` | `BigDecimal` | no | Debit amount; zero when line is a credit. |
| | `credit` | `BigDecimal` | no | Credit amount; zero when line is a debit. |
| | `postingDate` | `LocalDate` | no | Date the entry was posted to the GL. |
| `AccountBalance` | `accountId` | `String` | no | Chart-of-accounts code. |
| | `accountName` | `String` | no | Human-readable account name. |
| | `expectedBalance` | `BigDecimal` | no | Target closing balance for the reconciliation period. |
| | `currency` | `String` | no | ISO 4217 currency code. |
| `LedgerSnapshot` | `accountSetId` | `String` | no | Identifier of the account set being reconciled. |
| | `entries` | `List<LedgerEntry>` | no | Possibly empty. |
| | `expectedBalances` | `List<AccountBalance>` | no | One row per account in the set. |
| | `fetchedAt` | `Instant` | no | When the FETCH task returned. |
| `Variance` | `accountId` | `String` | no | Chart-of-accounts code. |
| | `expectedBalance` | `BigDecimal` | no | From the `AccountBalance` row. |
| | `actualBalance` | `BigDecimal` | no | Computed from the `LedgerEntry` rows. |
| | `delta` | `BigDecimal` | no | `actualBalance - expectedBalance`. |
| | `currency` | `String` | no | ISO 4217 currency code. |
| | `isMaterial` | `boolean` | no | True when `abs(delta) > materialThreshold`. |
| `NavCalculation` | `fundId` | `String` | no | Fund identifier derived from the account set. |
| | `totalAssets` | `BigDecimal` | no | Sum of all asset account balances. |
| | `totalLiabilities` | `BigDecimal` | no | Sum of all liability account balances. |
| | `nav` | `BigDecimal` | no | `totalAssets - totalLiabilities`. |
| | `calculatedAt` | `Instant` | no | When `ReconcileTools.calculateNAV` returned. |
| `ReconciliationResult` | `variances` | `List<Variance>` | no | One row per `AccountBalance` in the snapshot. |
| | `nav` | `Optional<NavCalculation>` | yes | Present when the account set includes fund accounts. |
| | `allAccountsReconciled` | `boolean` | no | True when all `Variance.delta == 0`. |
| | `hasMaterialVariance` | `boolean` | no | True when any `Variance.isMaterial == true`. |
| | `reconciledAt` | `Instant` | no | When the RECONCILE task returned. |
| `JournalLine` | `lineId` | `String` | no | Short stable id (`jl-<accountId>`). |
| | `accountId` | `String` | no | MUST be a code from `coa.json`. |
| | `debit` | `BigDecimal` | no | Debit amount; zero when line is a credit. |
| | `credit` | `BigDecimal` | no | Credit amount; zero when line is a debit. |
| | `description` | `String` | no | Narrative description. |
| | `reference` | `String` | no | Typically the `accountSetId`. |
| `JournalEntry` | `journalId` | `String` | no | Unique identifier for the journal entry. |
| | `lines` | `List<JournalLine>` | no | May be empty when all deltas are zero. |
| | `totalDebits` | `BigDecimal` | no | `sum(lines[*].debit)`. MUST equal `totalCredits` (G1 rule 1). |
| | `totalCredits` | `BigDecimal` | no | `sum(lines[*].credit)`. MUST equal `totalDebits` (G1 rule 1). |
| | `periodEnd` | `LocalDate` | no | Last day of the reconciliation period. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `ValidationResult` | `passed` | `boolean` | no | True when `PostingGuardrail` accepted the draft. |
| | `findings` | `List<String>` | no | Empty on pass; one entry per violated rule on fail. |
| | `validatedAt` | `Instant` | no | When the guardrail ran. |
| `EscalationRecord` | `escalationId` | `String` | no | Unique identifier. |
| | `runId` | `String` | no | The parent reconciliation run. |
| | `reason` | `String` | no | Names all material accounts and their deltas. |
| | `reviewedBy` | `Optional<String>` | yes | Controller identity, set on decision. |
| | `decision` | `Optional<String>` | yes | `APPROVED` or `REJECTED`, set on decision. |
| | `raisedAt` | `Instant` | no | When `EscalationRaised` was emitted. |
| | `decidedAt` | `Optional<Instant>` | yes | When `EscalationApproved` or `EscalationRejected` was emitted. |
| `ReconciliationRun` (entity state) | `runId` | `String` | no | — |
| | `accountSetId` | `Optional<String>` | yes | Populated after `RunCreated`. |
| | `snapshot` | `Optional<LedgerSnapshot>` | yes | Populated after `EntriesFetched`. |
| | `reconciliation` | `Optional<ReconciliationResult>` | yes | Populated after `ReconciliationCompleted`. |
| | `escalation` | `Optional<EscalationRecord>` | yes | Populated after `EscalationRaised`. |
| | `journal` | `Optional<JournalEntry>` | yes | Populated after `JournalDrafted`. |
| | `validation` | `Optional<ValidationResult>` | yes | Populated after `ValidationPassed`. |
| | `status` | `ReconciliationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `ReconciliationRun` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ReconciliationStatus`: `CREATED`, `FETCHING`, `FETCHED`, `RECONCILING`, `RECONCILED`, `PENDING_APPROVAL`, `DRAFTING`, `DRAFT_VALIDATED`, `POSTED`, `REJECTED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations): `FETCH`, `RECONCILE`, `DRAFT`.

## Events (`LedgerReconciliationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunCreated` | `accountSetId: String` | → CREATED |
| `FetchStarted` | — | → FETCHING |
| `EntriesFetched` | `snapshot: LedgerSnapshot` | → FETCHED |
| `ReconcileStarted` | — | → RECONCILING |
| `ReconciliationCompleted` | `reconciliation: ReconciliationResult` | → RECONCILED |
| `EscalationRaised` | `reason: String` | → PENDING_APPROVAL |
| `EscalationApproved` | `reviewedBy: String` | → DRAFTING (resumed from PENDING_APPROVAL) |
| `EscalationRejected` | `reviewedBy: String, reason: String` | → REJECTED (terminal) |
| `DraftStarted` | — | → DRAFTING |
| `JournalDrafted` | `journal: JournalEntry` | → DRAFTING (pending validation) |
| `ValidationPassed` | `validatedAt: Instant` | → DRAFT_VALIDATED |
| `ValidationFailed` | `findings: List<String>` | no status change (audit-only; agent retries) |
| `JournalPosted` | `postedAt: Instant` | → POSTED (terminal happy) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ReconciliationRun.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ReconciliationRow` mirrors `ReconciliationRun` exactly. The UI fetches the full row via `GET /api/reconciliations/{id}` and streams updates via `GET /api/reconciliations/sse`.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM reconciliation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`LedgerTasks.java`)

```java
public final class LedgerTasks {
  public static final Task<LedgerSnapshot> FETCH_ENTRIES = Task
      .name("Fetch entries")
      .description("Retrieve all ledger entries and expected account balances for an account set"
          + " by calling fetchLedgerEntries and fetchAccountBalances")
      .resultConformsTo(LedgerSnapshot.class);

  public static final Task<ReconciliationResult> RECONCILE_ACCOUNTS = Task
      .name("Reconcile accounts")
      .description("Compute per-account variances and calculate NAV"
          + " by calling computeVariances and calculateNAV")
      .resultConformsTo(ReconciliationResult.class);

  public static final Task<JournalEntry> DRAFT_JOURNAL = Task
      .name("Draft journal")
      .description("Compose a balanced journal entry whose lines offset each material variance")
      .resultConformsTo(JournalEntry.class);

  private LedgerTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## PostingGuardrail rules

`PostingGuardrail` is registered on `LedgerAgent` as a `before-agent-response` guardrail. It fires on every task result. For `FETCH_ENTRIES` and `RECONCILE_ACCOUNTS` results it passes immediately. For `DRAFT_JOURNAL` results it applies:

1. **Balance rule** — `totalDebits.compareTo(totalCredits) == 0`.
2. **Account-code rule** — every `JournalLine.accountId` is present in `ChartOfAccounts` loaded from `src/main/resources/sample-data/coa.json`.

On failure the guardrail returns a structured `posting-safety-violation` error to the agent loop and calls `LedgerReconciliationEntity.recordValidationFailed(findings)`. On accept it returns without side effects; the workflow then writes `ValidationPassed` and `JournalPosted`.
