# Data model — pgvector-rag-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Chunk` | `chunkId` | `String` | no | `documentId + "-" + index`. |
| | `documentId` | `String` | no | Parent document. |
| | `sourceLabel` | `String` | no | Human-readable corpus label. |
| | `text` | `String` | no | Chunk text (≈ 150 tokens). |
| | `tokenCount` | `int` | no | Approximate token count. |
| `RetrievedPassage` | `chunkId` | `String` | no | Chunk id from pgvector result. |
| | `sourceLabel` | `String` | no | Copied from the chunk row. |
| | `text` | `String` | no | Passage text. |
| | `relevanceScore` | `float` | no | Cosine similarity (0–1). |
| `Citation` | `chunkId` | `String` | no | MUST be a chunkId in the retrieved set. |
| | `sourceLabel` | `String` | no | Copied from the cited chunk. |
| `Answer` | `answerText` | `String` | no | 2–8 sentences with inline [chunkId] markers. |
| | `citations` | `List<Citation>` | no | One entry per cited chunk. Empty only for insufficient-context answers. |
| | `passages` | `List<RetrievedPassage>` | no | Full retrieved set for transparency. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `AnswerEvaluationScorer` finished. |
| `Question` (entity state) | `questionId` | `String` | no | — |
| | `questionText` | `String` | no | User-supplied question. |
| | `askedBy` | `String` | no | User identifier. |
| | `answer` | `Optional<Answer>` | yes | Populated after `AnswerRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `QuestionStatus` | no | See enum. |
| | `submittedAt` | `Instant` | no | When `QuestionSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `CorpusDocument` (entity state) | `documentId` | `String` | no | — |
| | `sourceLabel` | `String` | no | Human-readable label. |
| | `body` | `String` | no | Full document text. Kept on entity for reindexing. |
| | `status` | `CorpusStatus` | no | See enum. |
| | `addedAt` | `Instant` | no | When `DocumentAdded` emitted. |
| | `chunkCount` | `Optional<Integer>` | yes | Populated after `DocumentIndexed`. |

Every nullable field on `Question` and `CorpusDocument` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QuestionStatus`: `SUBMITTED`, `RETRIEVING`, `ANSWERING`, `ANSWERED`, `EVALUATED`, `FAILED`.

`CorpusStatus`: `ADDED`, `INDEXED`, `FAILED`.

## Events (`QuestionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuestionSubmitted` | `questionText`, `askedBy` | → SUBMITTED |
| `RetrievalStarted` | — | → RETRIEVING |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `answer` | → ANSWERED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `QuestionFailed` | `reason: String` | → FAILED (terminal) |

## Events (`CorpusEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DocumentAdded` | `sourceLabel`, `body` | → ADDED |
| `DocumentIndexed` | `chunkCount: int` | → INDEXED |
| `IndexingFailed` | `reason: String` | → FAILED |

`emptyState()` on `QuestionEntity` returns `Question.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QuestionRow` mirrors `Question` minus any raw large-text payload (the full `Answer.passages` list is on the entity for the detail view; the list view uses a summary). The view declares ONE query: `getAllQuestions: SELECT * AS questions FROM question_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## pgvector schema

The `corpus_chunks` table in the `ragagent` database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS corpus_chunks (
    chunk_id     TEXT PRIMARY KEY,
    document_id  TEXT NOT NULL,
    source_label TEXT NOT NULL,
    text         TEXT NOT NULL,
    embedding    vector(1536)
);

CREATE INDEX IF NOT EXISTS corpus_chunks_embedding_idx
    ON corpus_chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

`PgVectorStore` creates this schema on startup if it does not exist (using `CREATE TABLE IF NOT EXISTS` and `CREATE INDEX IF NOT EXISTS` — idempotent).

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<Answer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Read the retrieved passages and produce a grounded Answer citing chunk ids")
      .resultConformsTo(Answer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
