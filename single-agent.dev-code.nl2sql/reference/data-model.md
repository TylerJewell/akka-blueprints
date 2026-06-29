# Data model — nl2sql

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SchemaColumn` | `columnName` | `String` | no | Column name as it appears in the database. |
| | `dataType` | `String` | no | SQL data type (e.g., `uuid`, `text`, `numeric`, `timestamp`). |
| | `nullable` | `boolean` | no | Whether the column accepts NULL. |
| `SchemaTable` | `tableName` | `String` | no | Table name as it appears in the database. |
| | `description` | `String` | no | One-sentence description of the table's contents. |
| | `columns` | `List<SchemaColumn>` | no | Ordered column list. |
| `SchemaFragment` | `tables` | `List<SchemaTable>` | no | Tables relevant to this context. |
| | `contextName` | `String` | no | Name of the schema context (e.g., `orders`, `customers`, `fulfillment`, `full`). |
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `question` | `String` | no | User's natural-language question. |
| | `schemaContext` | `String` | no | Which schema context to use. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `QueryResult` | `generatedSql` | `String` | no | The exact SQL passed to `QueryExecutionTool`. |
| | `columnNames` | `List<String>` | no | Column headers in order. |
| | `rows` | `List<List<Object>>` | no | Result rows; each inner list is one row. |
| | `rowCount` | `int` | no | Number of rows returned. |
| | `summary` | `String` | no | 1–2 sentences describing what the result shows. |
| | `executedAt` | `Instant` | no | When `QueryExecutionTool` returned. |
| `ScoreResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| | `scoredAt` | `Instant` | no | When `QueryResultScorer` finished. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `schema` | `Optional<SchemaFragment>` | yes | Populated after `SchemaAttached`. |
| | `result` | `Optional<QueryResult>` | yes | Populated after `ResultRecorded`. |
| | `score` | `Optional<ScoreResult>` | yes | Populated after `QueryScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `SUBMITTED`, `SCHEMA_ATTACHED`, `TRANSLATING`, `RESULT_READY`, `SCORED`, `HALTED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `SchemaAttached` | `schema` | → SCHEMA_ATTACHED |
| `TranslationStarted` | — | → TRANSLATING |
| `ResultRecorded` | `result` | → RESULT_READY |
| `QueryScored` | `score` | → SCORED (terminal happy) |
| `QueryHalted` | `reason: String` | → HALTED (terminal — safety halt fired) |
| `QueryFailed` | `reason: String` | → FAILED (terminal — infrastructure or agent error) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query` directly. No sensitive raw content is excluded (queries carry questions and SQL, not PII). The UI fetches the full row via `GET /api/queries/{id}`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<QueryResult> RUN_QUERY = Task
      .name("Run query")
      .description("Translate the natural-language question to SQL, execute it against "
          + "the Postgres database, and return a QueryResult")
      .resultConformsTo(QueryResult.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Schema registry (in-process seed)

`SchemaRegistry` is seeded from `src/main/resources/sample-events/schema-registry.jsonl`. Each line is a JSON object with `contextName` and `tables`. Registered contexts:

| Context | Tables |
|---|---|
| `orders` | `orders`, `order_items`, `products` |
| `customers` | `customers`, `orders` |
| `fulfillment` | `fulfillment`, `orders`, `products` |
| `full` | All five tables |

The `SchemaLookupTool` used by `SqlQueryAgent` delegates to `SchemaRegistry.lookup(tableName)` which returns a single `SchemaTable` definition.
