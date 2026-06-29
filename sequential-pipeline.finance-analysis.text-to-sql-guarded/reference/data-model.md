# Data model — text-to-sql-guarded

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SchemaTable` | `tableName` | `String` | no | Exact H2 table name as declared in seed.sql. |
| | `columnNames` | `List<String>` | no | All projected columns for this table. |
| | `description` | `String` | no | One-line description loaded from schema/*.json. |
| `ParsedQuery` | `question` | `String` | no | The original user question. |
| | `sql` | `String` | no | Generated SQL SELECT statement (or `-- no relevant tables found`). |
| | `tablesReferenced` | `List<SchemaTable>` | no | Possibly empty when no schema match. |
| | `parsedAt` | `Instant` | no | When the PARSE task returned. |
| `ResultRow` | `cells` | `Map<String,String>` | no | Column name → cell value (as String). PII-tagged columns carry raw values in the raw result; redacted values in the sanitized result. |
| `RawResult` | `columns` | `List<String>` | no | Projected column names in order. |
| | `rows` | `List<ResultRow>` | no | Possibly empty. Raw, un-redacted values. |
| | `rowCount` | `int` | no | `rows.size()`. |
| | `queriedAt` | `Instant` | no | When the QUERY task returned. |
| `RedactionSummary` | `redactionCount` | `int` | no | Total number of cells redacted across all rows. |
| | `redactedFields` | `List<String>` | no | Column names where at least one cell was redacted. |
| `SanitizedResult` | `columns` | `List<String>` | no | Same column list as `RawResult`. |
| | `rows` | `List<ResultRow>` | no | Rows with PII cells replaced. |
| | `redactionSummary` | `RedactionSummary` | no | Audit summary of what was masked. |
| `QueryReport` | `question` | `String` | no | The original user question. |
| | `sqlUsed` | `String` | no | The SQL that was executed. |
| | `narrative` | `String` | no | 1–3 sentence plain-English summary from the FORMAT task. |
| | `result` | `SanitizedResult` | no | Sanitized result set embedded in the report. |
| | `formattedAt` | `Instant` | no | When the FORMAT task returned. |
| `SafetyDecision` | `safe` | `boolean` | no | `true` if no destructive pattern found. |
| | `matchedPattern` | `Optional<String>` | yes | The first matched keyword if `safe = false`. |
| `GuardrailRejection` | `phase` | `String` | no | `PARSE` / `QUERY` / `FORMAT`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `SqlGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `QueryRecord` (entity state) | `queryId` | `String` | no | — |
| | `question` | `Optional<String>` | yes | Populated after `QueryCreated`. |
| | `parsedQuery` | `Optional<ParsedQuery>` | yes | Populated after `QuestionParsed`. |
| | `rawResult` | `Optional<RawResult>` | yes | Populated after `QueryExecuted`. Not returned in public API. |
| | `report` | `Optional<QueryReport>` | yes | Populated after `ResultsFormatted`. |
| | `haltReason` | `Optional<String>` | yes | Populated after `SafetyHaltFired`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `QueryRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `CREATED`, `PARSING`, `PARSED`, `QUERYING`, `QUERIED`, `FORMATTING`, `FORMATTED`, `HALTED`, `FAILED`.

`SqlPhase` (used by `@FunctionTool` annotations and `SqlGuardrail`): `PARSE`, `QUERY`, `FORMAT`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QueryCreated` | `question: String` | → CREATED |
| `ParseStarted` | — | → PARSING |
| `QuestionParsed` | `parsedQuery: ParsedQuery` | → PARSED |
| `SafetyHaltFired` | `sql: String, matchedPattern: String` | → HALTED (terminal) |
| `QueryStarted` | — | → QUERYING |
| `QueryExecuted` | `rawResult: RawResult` | → QUERIED |
| `FormatStarted` | — | → FORMATTING |
| `ResultsFormatted` | `report: QueryReport` | → FORMATTED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `QueryRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `QueryRecord` exactly, **except** that `rawResult` is excluded from the public API response. The UI fetches the full row via `GET /api/queries/{id}` and streams updates via `GET /api/queries/sse`. The `rawResult` field is stored on the entity for the internal audit trail but stripped by `QueryEndpoint` before serialising the response.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`SqlTasks.java`)

```java
public final class SqlTasks {
  public static final Task<ParsedQuery> PARSE_QUESTION = Task
      .name("Parse question")
      .description("Resolve the financial schema and generate a read-only SQL statement for the user question")
      .resultConformsTo(ParsedQuery.class);

  public static final Task<RawResult> EXECUTE_QUERY = Task
      .name("Execute query")
      .description("Execute the validated SQL against the financial database and return the raw result set")
      .resultConformsTo(RawResult.class);

  public static final Task<QueryReport> FORMAT_RESULTS = Task
      .name("Format results")
      .description("Produce a narrative summary and formatted table from the sanitized result set")
      .resultConformsTo(QueryReport.class);

  private SqlTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ParseTools`, `QueryTools`, and `FormatTools` carries a `SqlPhase` constant. `SqlGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G1). The tool registry is built once at startup; the guardrail reads it for every call.

## Safety inspection

`SqlSafetyInspector` is invoked by `QueryPipelineWorkflow.parseStep` after the agent returns `ParsedQuery` and before `QueryEntity.recordParsed` is called. Its output is `SafetyDecision`. No Akka primitive — it is a stateless, synchronous Java class with no external dependencies. The inspected SQL is the raw string from `ParsedQuery.sql`; the inspector does not parse AST — it applies word-boundary regex to the entire string.

## PII sanitization

`PiiSanitizer` is invoked by `QueryPipelineWorkflow.formatStep` on the `RawResult` before `runSingleTask(FORMAT_RESULTS)`. Its output is `SanitizedResult`. The three redaction rules (SSN regex, account-number column match, name lookup) are applied in order; a cell that matches multiple rules is redacted by the first matching rule only. The `RedactionSummary` counts each redacted cell once regardless of how many rules matched.
