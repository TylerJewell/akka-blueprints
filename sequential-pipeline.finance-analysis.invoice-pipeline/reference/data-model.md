# Data model — invoice-processing

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InvoiceHeader` | `invoiceNumber` | `String` | no | Vendor-assigned invoice identifier. |
| | `vendorName` | `String` | no | Vendor's trading name. |
| | `vendorEmail` | `String` | no | Contact email — always `"[REDACTED]"` after `VendorPiiSanitizer` fires. |
| | `vendorPhone` | `String` | no | Contact phone — always `"[REDACTED]"` after `VendorPiiSanitizer` fires. |
| | `invoiceDate` | `LocalDate` | no | Date on the invoice document. |
| | `currency` | `String` | no | ISO 4217 currency code (e.g. `USD`). |
| | `totalAmount` | `BigDecimal` | no | Declared invoice total; used by the HITL threshold check. |
| `LineItem` | `lineId` | `String` | no | Short sequential id (`li-1`, `li-2`, …). |
| | `description` | `String` | no | Line-item description from the invoice document. |
| | `quantity` | `int` | no | Positive integer quantity. |
| | `unitPrice` | `BigDecimal` | no | Per-unit price in the invoice currency. |
| | `lineTotal` | `BigDecimal` | no | `quantity × unitPrice`. |
| `ExtractedInvoice` | `header` | `InvoiceHeader` | no | Parsed and sanitized header. |
| | `lineItems` | `List<LineItem>` | no | Possibly empty for unrecognised formats. |
| | `extractedAt` | `Instant` | no | When the EXTRACT task returned. |
| `SanitizationReport` | `redactedFields` | `List<String>` | no | Field names redacted (`["vendorEmail","vendorPhone"]`). No original values. |
| | `sanitizedAt` | `Instant` | no | When `VendorPiiSanitizer` ran. |
| `GlAccount` | `accountCode` | `String` | no | GL account number (e.g. `"6200"`). |
| | `accountName` | `String` | no | Human-readable account name. |
| | `accountType` | `String` | no | `EXPENSE`, `LIABILITY`, `ASSET`, etc. |
| `ValidatedLineItem` | `lineItem` | `LineItem` | no | The original line item. |
| | `glAccount` | `GlAccount` | no | Assigned GL account from `lookupGlAccount`. |
| | `balanceOk` | `boolean` | no | `true` when `lineItem.lineTotal` arithmetic holds. |
| `BalanceResult` | `balanced` | `boolean` | no | `true` when debitTotal equals creditTotal. |
| | `debitTotal` | `BigDecimal` | no | Sum of all expense-side amounts. |
| | `creditTotal` | `BigDecimal` | no | Sum of all AP-side amounts. |
| `ValidatedInvoice` | `source` | `ExtractedInvoice` | no | The sanitized extracted invoice. |
| | `lineItems` | `List<ValidatedLineItem>` | no | One per line item. |
| | `balance` | `BalanceResult` | no | Overall balance check. |
| | `totalAmount` | `BigDecimal` | no | Equals `InvoiceHeader.totalAmount`; used by the HITL gate. |
| | `validatedAt` | `Instant` | no | When the VALIDATE task returned. |
| `JournalLine` | `accountCode` | `String` | no | GL account number. |
| | `accountName` | `String` | no | GL account name. |
| | `debit` | `BigDecimal` | no | Debit amount; `0` for credit lines. |
| | `credit` | `BigDecimal` | no | Credit amount; `0` for debit lines. |
| `JournalEntry` | `entryId` | `String` | no | System-minted id. |
| | `invoiceRef` | `String` | no | Equals `InvoiceHeader.invoiceNumber`. |
| | `lines` | `List<JournalLine>` | no | One debit per expense line item, one AP credit. |
| | `entryDate` | `LocalDate` | no | Date the entry was built. |
| `PostingConfirmation` | `confirmationRef` | `String` | no | `"conf-" + sha1(entryId).substring(0,8)`. |
| | `postedAt` | `Instant` | no | When the in-process ledger stub accepted the entry. |
| `PostedInvoice` | `validated` | `ValidatedInvoice` | no | The validated invoice that drove the posting. |
| | `journalEntry` | `JournalEntry` | no | The posted journal entry. |
| | `confirmation` | `PostingConfirmation` | no | Ledger confirmation. |
| | `postedAt` | `Instant` | no | When the POST task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `PostingQualityScorer` finished. |
| `InvoiceRecord` (entity state) | `invoiceId` | `String` | no | — |
| | `rawText` | `Optional<String>` | yes | Populated after `InvoiceCreated`. |
| | `extracted` | `Optional<ExtractedInvoice>` | yes | Populated after `ExtractionCompleted`. |
| | `sanitization` | `Optional<SanitizationReport>` | yes | Populated after `SanitizationApplied`. |
| | `validated` | `Optional<ValidatedInvoice>` | yes | Populated after `ValidationCompleted`. |
| | `posted` | `Optional<PostedInvoice>` | yes | Populated after `PostingCompleted`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `InvoiceStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `InvoiceCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `InvoiceRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`InvoiceStatus`: `CREATED`, `EXTRACTING`, `EXTRACTED`, `VALIDATING`, `VALIDATED`, `APPROVAL_REQUESTED`, `APPROVED`, `POSTING`, `POSTED`, `EVALUATED`, `DENIED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `PhaseGuardrail`): `EXTRACT`, `VALIDATE`, `POST`.

## Events (`InvoiceEntity`)

| Event | Payload | Transition |
|---|---|---|
| `InvoiceCreated` | `rawText: String` | → CREATED |
| `ExtractionStarted` | — | → EXTRACTING |
| `SanitizationApplied` | `redactedFields: List<String>, sanitizedAt: Instant` | no status change (audit; always precedes ExtractionCompleted) |
| `ExtractionCompleted` | `extracted: ExtractedInvoice` | → EXTRACTED |
| `ValidationStarted` | — | → VALIDATING |
| `ValidationCompleted` | `validated: ValidatedInvoice` | → VALIDATED |
| `ApprovalRequested` | `totalAmount: BigDecimal` | → APPROVAL_REQUESTED |
| `ApprovalGranted` | `reviewerNote: String` | → APPROVED |
| `ApprovalDenied` | `reviewerNote: String` | → DENIED (terminal) |
| `PostingStarted` | — | → POSTING |
| `PostingCompleted` | `posted: PostedInvoice` | → POSTED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `PhaseGuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `InvoiceFailed` | `reason: String` | → FAILED (terminal) |

`SanitizationApplied` is always written BEFORE `ExtractionCompleted` within the same `extractStep`. This ordering in the event log is the auditable proof that PII was removed before the extract data was committed. `emptyState()` returns `InvoiceRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`InvoiceRow` mirrors `InvoiceRecord` exactly. The UI fetches the full row via `GET /api/invoices/{id}` and streams updates via `GET /api/invoices/sse`.

The view declares ONE query: `getAllInvoices: SELECT * AS invoices FROM invoice_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`InvoiceTasks.java`)

```java
public final class InvoiceTasks {
  public static final Task<ExtractedInvoice> EXTRACT_INVOICE = Task
      .name("Extract invoice")
      .description("Parse the invoice header and all line items from the raw text")
      .resultConformsTo(ExtractedInvoice.class);

  public static final Task<ValidatedInvoice> VALIDATE_LINE_ITEMS = Task
      .name("Validate line items")
      .description("Look up the GL account for each line item and check overall balance")
      .resultConformsTo(ValidatedInvoice.class);

  public static final Task<PostedInvoice> POST_JOURNAL_ENTRY = Task
      .name("Post journal entry")
      .description("Build a balanced journal entry from the validated invoice and confirm the posting")
      .resultConformsTo(PostedInvoice.class);

  private InvoiceTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ExtractTools`, `ValidateTools`, and `PostTools` carries a `Phase` constant. `PhaseGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml — `PhaseGuardrail` enforces the sequential-pipeline property). The tool registry is built once at startup; the guardrail reads it for every call.

## PostingQualityScorer checks

| Check | Points | Passes when |
|---|---|---|
| GL coverage | 1 | Every `ValidatedLineItem.glAccount.accountCode` appears in at least one `JournalLine.accountCode` |
| Journal balance | 1 | `sum(JournalLine.debit) == sum(JournalLine.credit)` |
| Line parity | 1 | `journalEntry.lines.size() >= validatedInvoice.lineItems.size()` |
| Confirmation present | 1 | `PostingConfirmation.confirmationRef` is non-blank |

Score = 1 (base) + number of checks that pass. Range 1–5. Rationale names the first failing check.
