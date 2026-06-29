# Data model — text-to-sql-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `question` | `String` | no | Natural-language question as submitted. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `GeneratedSql` | `sql` | `String` | no | The SELECT statement produced by the agent. |
| | `explanation` | `String` | no | One sentence describing why this SQL answers the question. |
| `ResultRow` | `columns` | `Map<String, Object>` | no | Column-name-to-value map for one database row. |
| `RawQueryResult` | `generatedSql` | `GeneratedSql` | no | The SQL that was executed. |
| | `rows` | `List<ResultRow>` | no | Raw rows from SQLite. May contain un-redacted PII. Audit-only. |
| | `summary` | `String` | no | One-sentence prose answer. |
| | `executedAt` | `Instant` | no | When the tool call returned. |
| `SanitizedResult` | `rows` | `List<ResultRow>` | no | Rows with customer names and emails replaced by redaction tokens. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["customer-name", "email"]`. Empty list if none found. |
| `QueryResult` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuestionSubmitted`. |
| | `generatedSql` | `Optional<GeneratedSql>` | yes | Populated after `SqlGenerated`. |
| | `rawResult` | `Optional<RawQueryResult>` | yes | Populated after `QueryExecuted`. |
| | `sanitized` | `Optional<SanitizedResult>` | yes | Populated after `ResultSanitized`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuestionSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `QueryResult` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `SUBMITTED`, `GENERATING`, `EXECUTING`, `EXECUTED`, `SANITIZED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuestionSubmitted` | `request: QueryRequest` | → SUBMITTED |
| `SqlGenerated` | `sql: String`, `explanation: String` | → GENERATING |
| `ExecutionStarted` | — | → EXECUTING |
| `QueryExecuted` | `rawResult: RawQueryResult` | → EXECUTED |
| `ResultSanitized` | `sanitized: SanitizedResult` | → SANITIZED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `QueryResult.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `QueryResult` minus `rawResult.rows` (the audit log keeps raw rows on the entity). The view carries `rawResult.generatedSql`, `rawResult.summary`, and `sanitized` (the redacted rows). The full `rawResult.rows` are accessible only via `GET /api/queries/{id}` on the endpoint.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint sorts client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<QueryResult> GENERATE_AND_RUN_QUERY = Task
      .name("Generate and run SQL query")
      .description("Translate the natural-language question into SQL, execute it against the receipts table, and return the result rows with a prose summary.")
      .resultConformsTo(QueryResult.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## SQLite schema (`receipts.db`)

```sql
CREATE TABLE receipts (
  receipt_id     TEXT PRIMARY KEY,
  customer_name  TEXT,
  customer_email TEXT,
  merchant       TEXT,
  amount_cents   INTEGER,
  currency       TEXT,
  receipt_date   TEXT,   -- ISO-8601 date (YYYY-MM-DD)
  category       TEXT
);
```

The database is pre-seeded with 50 rows covering 10 merchants (`Acme Travel`, `QuickBite`, `CloudSoft Inc`, `OfficeHub`, `FlyFast`, `DataDine`, `MeetCo`, `PrintWorks`, `SafeHosting`, `GreenFleet`), 8 customers, 4 categories (`Food & Beverage`, `Travel`, `Office Supplies`, `Software`), and receipt dates from 2025-01-01 through 2026-06-28. Amounts range from $8.50 to $1,200.00. The `schema.sql` file (identical DDL) is the attachment passed to `SqlGeneratorAgent`.
