# Data model — rag-citation-answerer

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `UploadRequest` | `documentId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `documentTitle` | `String` | no | User-supplied label. |
| | `rawText` | `String` | no | Pre-sanitization document body. Audit-only. |
| | `uploadedBy` | `String` | no | User identifier. |
| | `uploadedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedText` | `redactedText` | `String` | no | PII redacted; this is the source for chunking. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","ssn","payment-card-number","address","person-name"]`. |
| `Chunk` | `chunkId` | `String` | no | `"doc-" + documentId + "-" + chunkIndex`. |
| | `documentId` | `String` | no | Parent document. |
| | `documentTitle` | `String` | no | Copied from the parent document for display. |
| | `chunkIndex` | `int` | no | Zero-based position within the document. |
| | `text` | `String` | no | Chunk content (512-char max, 64-char overlap). |
| | `sectionHint` | `String` | no | Parsed section marker, or empty string. |
| `RetrievalResult` | `chunks` | `List<Chunk>` | no | Top-K chunks returned by `ChunkStore`. |
| | `totalChunksSearched` | `int` | no | Count of candidate chunks before top-K filter. |
| `Citation` | `chunkId` | `String` | no | MUST match a `chunkId` in the task attachment. |
| | `documentId` | `String` | no | Copied from the chunk. |
| | `documentTitle` | `String` | no | Copied from the chunk. |
| | `excerpt` | `String` | no | Verbatim passage from the chunk's text. |
| | `sectionHint` | `String` | no | Copied from the chunk; may be empty string. |
| `CitedAnswer` | `answerText` | `String` | no | 1–5 sentences. |
| | `confidence` | `Confidence` | no | Enum value. |
| | `citations` | `List<Citation>` | no | Per-chunk references drawn on for the answer. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `Document` (entity state) | `documentId` | `String` | no | — |
| | `upload` | `Optional<UploadRequest>` | yes | Populated after `DocumentUploaded`. |
| | `sanitized` | `Optional<SanitizedText>` | yes | Populated after `DocumentSanitized`. |
| | `status` | `DocumentStatus` | no | See enum. |
| | `chunkCount` | `int` | no | 0 until `DocumentIndexed`. |
| | `createdAt` | `Instant` | no | When `DocumentUploaded` emitted. |
| | `indexedAt` | `Optional<Instant>` | yes | Set when `DocumentIndexed` emitted. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `questionText` | `String` | no | Populated after `QuerySubmitted`. |
| | `documentIds` | `List<String>` | no | Populated after `QuerySubmitted`. |
| | `askedBy` | `String` | no | Populated after `QuerySubmitted`. |
| | `retrieval` | `Optional<RetrievalResult>` | yes | Populated after `RetrievalStarted`. |
| | `answer` | `Optional<CitedAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `answeredAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Document` and `Query` is `Optional<T>`. View table updaters wrap values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Confidence`: `HIGH`, `MEDIUM`, `LOW`.

`DocumentStatus`: `UPLOADED`, `SANITIZED`, `CHUNKED`, `INDEXED`, `FAILED`.

`QueryStatus`: `SUBMITTED`, `RETRIEVING`, `ANSWERED`, `FAILED`.

## Events (`DocumentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DocumentUploaded` | `upload: UploadRequest` | → UPLOADED |
| `DocumentSanitized` | `sanitized: SanitizedText` | → SANITIZED |
| `DocumentChunked` | `chunkCount: int` | → CHUNKED |
| `DocumentIndexed` | `chunkCount: int` | → INDEXED (terminal happy) |
| `DocumentFailed` | `reason: String` | → FAILED (terminal) |

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `questionText`, `documentIds`, `askedBy` | → SUBMITTED |
| `RetrievalStarted` | `retrievedChunkCount: int` | → RETRIEVING |
| `AnswerRecorded` | `answer: CitedAnswer` | → ANSWERED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` on both entities returns the initial state record with all `Optional` fields as `Optional.empty()`, numeric fields as `0`, and list fields as `List.of()`. Neither `emptyState()` references `commandContext()` (Lesson 3).

## View rows

**`DocumentRow`** mirrors `Document` minus `upload.rawText` (the audit log keeps that field on the entity). `chunkCount` and `indexedAt` are included; they are updated as `DocumentChunked` and `DocumentIndexed` events land.

**`QueryRow`** mirrors `Query`. The full `retrieval` and `answer` fields are included so the UI can render citations without a second fetch.

Each view declares ONE query with no `WHERE` status filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint and UI filter client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<CitedAnswer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Read the attached document chunks and produce a CitedAnswer with grounded citations")
      .resultConformsTo(CitedAnswer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## ChunkStore (supporting class)

`ChunkStore` is an application-scoped singleton (`@Bean`). It wraps a `ConcurrentHashMap<String, Chunk>` keyed by `chunkId`. The `retrieve(queryId, documentIds, questionText, topK)` method:
1. Filters chunks whose `documentId` is in the provided `documentIds` set.
2. Scores each candidate chunk by keyword overlap (term-frequency intersection) between `chunk.text` and `questionText`.
3. Returns the top-K chunks by score as a `RetrievalResult`.

No external service, no embedding model, no network call. This is intentional for the out-of-the-box tier.
