# Data model — notion-rag

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `DatabaseSchema` | `databaseId` | `String` | no | Notion database identifier. |
| | `databaseName` | `String` | no | Human-readable name of the bound database. |
| | `properties` | `List<PropertyDefinition>` | no | Ordered list of column definitions. |
| `PropertyDefinition` | `propertyId` | `String` | no | Notion-assigned property ID. |
| | `name` | `String` | no | Display name of the column. |
| | `type` | `String` | no | Notion property type: `title`, `rich_text`, `number`, `select`, `multi_select`, `checkbox`, `url`, `date`. |
| `NotionRow` | `rowId` | `String` | no | Notion page ID used as the row key. |
| | `properties` | `Map<String, String>` | no | Property name → rendered string value for this row. |
| `RetrievedRows` | `databaseId` | `String` | no | Which database was queried. |
| | `queryText` | `String` | no | The question text used as the query. |
| | `rows` | `List<NotionRow>` | no | Rows returned by the Notion API (may be empty). |
| | `totalRowsFetched` | `int` | no | Count of rows in this batch. |
| `SourceCitation` | `rowId` | `String` | no | MUST equal a `rowId` in the `RetrievedRows`. |
| | `propertyName` | `String` | no | The property name being cited. |
| | `excerpt` | `String` | no | The property value text. |
| | `matchReason` | `String` | no | One phrase explaining relevance. |
| `QueryAnswer` | `answer` | `String` | no | Direct answer text, 1–4 sentences. |
| | `confidence` | `AnswerConfidence` | no | Enum value. |
| | `citations` | `List<SourceCitation>` | no | One entry per supporting row+property. Empty only when `confidence = NO_DATA`. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `Question` | `questionId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `sessionId` | `String` | no | Parent session. |
| | `questionText` | `String` | no | User-supplied question. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| | `retrievedRows` | `Optional<RetrievedRows>` | yes | Populated after `RowsRetrieved`. |
| | `answer` | `Optional<QueryAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `status` | `QuestionStatus` | no | See enum. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `label` | `String` | no | User-supplied or empty string. |
| | `questions` | `List<Question>` | no | All questions in order; empty list initially. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `lastActivityAt` | `Optional<Instant>` | yes | Updated on each new question or answer. |

Every nullable field uses `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AnswerConfidence`: `HIGH`, `MEDIUM`, `LOW`, `NO_DATA`.
`QuestionStatus`: `SUBMITTED`, `ROWS_RETRIEVED`, `ANSWERING`, `ANSWERED`, `FAILED`.
`SessionStatus`: `ACTIVE`, `CLOSED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `label`, `createdBy` | Creates session in `ACTIVE` status |
| `QuestionSubmitted` | `questionId`, `questionText`, `submittedBy`, `submittedAt` | Appends question at `SUBMITTED` |
| `RowsRetrieved` | `questionId`, `retrievedRows` | Question → `ROWS_RETRIEVED` |
| `AnswerStarted` | `questionId` | Question → `ANSWERING` |
| `AnswerRecorded` | `questionId`, `answer` | Question → `ANSWERED` (terminal happy) |
| `QuestionFailed` | `questionId`, `reason: String` | Question → `FAILED` (terminal) |

`emptyState()` returns `Session.initial("")` with `questions = List.of()`, `status = ACTIVE`, `lastActivityAt = Optional.empty()`, and no `commandContext()` reference (Lesson 3).

## View row

`SessionRow` mirrors `Session` but the `questions` list uses `QuestionSummary` (omits `retrievedRows.rows` body — the full row list is fetchable via `GET /api/sessions/{sessionId}` only). The UI fetches individual sessions on demand.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<QueryAnswer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Read the attached Notion rows and produce a QueryAnswer for the submitted question")
      .resultConformsTo(QueryAnswer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
