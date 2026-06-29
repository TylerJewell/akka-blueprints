# Data model — personal-finance-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `AccountRef` | `accountId` | `String` | no | Stable account identifier (may be tokenised in sanitized context). |
| | `label` | `String` | no | Human-readable name ("Main Checking"). |
| | `type` | `String` | no | `CHECKING`, `SAVINGS`, `CREDIT`, or `BUSINESS`. |
| `TransactionRecord` | `transactionId` | `String` | no | Stable identifier. |
| | `accountId` | `String` | no | Owning account. |
| | `rawDescription` | `String` | no | Pre-sanitization merchant description. Audit-only. |
| | `merchant` | `String` | no | Normalised merchant name. |
| | `category` | `String` | no | Spending category (Groceries, Dining, Utilities, etc.). |
| | `amount` | `BigDecimal` | no | Signed (negative = debit, positive = credit). |
| | `currency` | `String` | no | ISO-4217 code (e.g., `USD`). |
| | `date` | `LocalDate` | no | Transaction posting date. |
| `SanitizedTransaction` | `transactionId` | `String` | no | Same id as `TransactionRecord`. |
| | `accountId` | `String` | no | Tokenised form (e.g., `[ACCT-001]`). |
| | `description` | `String` | no | PII-redacted description. |
| | `merchant` | `String` | no | Same as raw (merchant name is not PII). |
| | `category` | `String` | no | Same as raw. |
| | `amount` | `BigDecimal` | no | Same as raw. |
| | `currency` | `String` | no | Same as raw. |
| | `date` | `LocalDate` | no | Same as raw. |
| `SanitizedContext` | `accounts` | `List<AccountRef>` | no | Account list with tokenised `accountId` values. |
| | `transactions` | `List<SanitizedTransaction>` | no | Sanitized transaction list. |
| | `piiTokensReplaced` | `int` | no | Total count of replacements across all fields. |
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `queryText` | `String` | no | User's natural-language question. |
| | `accountSetId` | `String` | no | Identifies the seeded account set. |
| | `rawTransactions` | `List<TransactionRecord>` | no | Pre-sanitization. Audit-only. |
| | `accounts` | `List<AccountRef>` | no | Raw account list (untokenised). |
| | `principal` | `String` | no | User identifier from the request. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCallRecord` | `toolName` | `String` | no | One of the five tool names. |
| | `arguments` | `String` | no | JSON-serialised tool arguments. |
| | `result` | `String` | no | JSON-serialised result or block reason. |
| | `outcome` | `WriteOutcome` | no | Enum value. |
| | `calledAt` | `Instant` | no | When the tool call occurred. |
| `AssistantResponse` | `answer` | `String` | no | Plain-language answer, 1–4 sentences. |
| | `structuredData` | `String` | yes | JSON table string, or `null`. |
| | `toolTrace` | `List<ToolCallRecord>` | no | One entry per tool invocation (may be empty). |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `sanitized` | `Optional<SanitizedContext>` | yes | Populated after `TransactionsSanitized`. |
| | `response` | `Optional<AssistantResponse>` | yes | Populated after `AnswerRecorded`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`WriteOutcome`: `NOT_A_WRITE`, `APPROVED`, `BLOCKED`, `ERROR`.

`QueryStatus`: `SUBMITTED`, `SANITIZED`, `ANSWERING`, `ANSWERED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `TransactionsSanitized` | `sanitized` | → SANITIZED |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `response` | → ANSWERED |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query` minus `request.rawTransactions` (the audit log keeps that field). The UI fetches the raw transaction list on demand via `GET /api/queries/{id}` and reads `request.rawTransactions` from the JSON.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`FinanceTasks.java`)

```java
public final class FinanceTasks {
  public static final Task<AssistantResponse> ANSWER_FINANCE_QUERY = Task
      .name("Answer finance query")
      .description("Use the available finance tools to answer the user's question and return an AssistantResponse")
      .resultConformsTo(AssistantResponse.class);

  private FinanceTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Tool definitions (`FinanceTools`)

| Tool | Arguments | Return type | Write? |
|---|---|---|---|
| `getBalance` | `accountId: String` | `BalanceResult { accountId, available, currency }` | no |
| `listTransactions` | `accountId: String, fromDate: LocalDate, toDate: LocalDate` | `List<SanitizedTransaction>` | no |
| `groupByCategory` | `transactions: List<SanitizedTransaction>` | `List<CategoryTotal { category, total, count }>` | no |
| `transferFunds` | `fromAccountId: String, toAccountId: String, amount: BigDecimal, note: String` | `TransferResult { transactionId, status }` | **yes** |
| `payBill` | `accountId: String, payee: String, amount: BigDecimal, reference: String` | `BillResult { confirmationId, status }` | **yes** |

Write tools (`transferFunds`, `payBill`) are registered with a metadata flag that the `WriteGuardrail` hook uses to identify them without a runtime name-check inside the tool body.
