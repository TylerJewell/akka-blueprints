# Data model — code-search-rag-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CodeChunk` | `chunkId` | `String` | no | Stable id assigned by the retrieval layer. |
| | `filePath` | `String` | no | Repository-relative path (e.g., `src/main/java/com/example/App.java`). |
| | `startLine` | `int` | no | First line of this chunk in the source file (1-based). |
| | `endLine` | `int` | no | Last line of this chunk in the source file (inclusive). |
| | `language` | `String` | no | Programming language tag (e.g., `java`, `scala`, `kotlin`). |
| | `content` | `String` | no | Source text of the chunk. May contain secrets before sanitization. |
| | `corpusTag` | `String` | no | Repository partition label (e.g., `akka-http`, `akka-streams`, `all`). |
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `questionText` | `String` | no | Developer's natural-language question. |
| | `corpusTag` | `String` | no | Filter applied at retrieval time. |
| | `retrievedChunks` | `List<CodeChunk>` | no | Raw chunks before sanitization. Audit-only. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedChunks` | `redactedChunks` | `List<CodeChunk>` | no | Chunks whose `content` has secrets replaced; same metadata as originals. |
| | `secretCategoriesFound` | `List<String>` | no | e.g., `["api-key", "pem-key", "conn-string"]`. Empty when none found. |
| `CodeReference` | `filePath` | `String` | no | MUST equal a `filePath` in the sanitized chunks set. |
| | `startLine` | `int` | no | From the cited chunk's `startLine`. |
| | `endLine` | `int` | no | From the cited chunk's `endLine`. |
| | `language` | `String` | no | From the cited chunk's `language`. |
| | `relevanceBlurb` | `String` | no | One sentence explaining relevance. |
| `SearchAnswer` | `answerText` | `String` | no | 1–4 sentence prose answer. |
| | `references` | `List<CodeReference>` | no | One entry per distinct cited file. May be empty if no chunks matched. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `GroundingResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `GroundingScorer` finished. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `sanitized` | `Optional<SanitizedChunks>` | yes | Populated after `ChunksSanitized`. |
| | `answer` | `Optional<SearchAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `grounding` | `Optional<GroundingResult>` | yes | Populated after `GroundingScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `SUBMITTED`, `CHUNKS_SANITIZED`, `ANSWERING`, `ANSWERED`, `GROUNDED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `ChunksSanitized` | `sanitized` | → CHUNKS_SANITIZED |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `answer` | → ANSWERED |
| `GroundingScored` | `grounding` | → GROUNDED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query` minus `request.retrievedChunks[*].content` (the raw chunk bodies stay on the entity for audit). The UI fetches raw content on demand via `GET /api/queries/{id}`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM chunk_index_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<SearchAnswer> ANSWER_CODE_QUESTION = Task
      .name("Answer code question")
      .description("Read the attached code chunks and return a SearchAnswer with file paths and line numbers")
      .resultConformsTo(SearchAnswer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Grounding scorer rubric (`GroundingScorer.java`)

| Condition | Points |
|---|---|
| `answerText` is non-empty | base |
| `references` is empty AND sanitized chunk list was empty | score = 5 (graceful empty result) |
| `references` is empty AND sanitized chunk list was non-empty | score = 1 (no citations despite available chunks) |
| Each `reference.filePath` found in `sanitized.redactedChunks[*].filePath` | 0 deducted |
| Each `reference.filePath` NOT found in `sanitized.redactedChunks[*].filePath` | −1 per ungrounded citation, floor 1 |

The scorer starts at 5 and applies deductions. Its output is always in `[1, 5]`.
