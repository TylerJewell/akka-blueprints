# Data model — rag-pdf-chat

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PdfPassage` | `passageId` | `String` | no | Assigned by indexer; e.g., `"P-001"`. |
| | `pageNumber` | `int` | no | Page in the source PDF (1-based). |
| | `chunkIndex` | `int` | no | Chunk position within the page. |
| | `text` | `String` | no | The passage text, ~300 tokens. |
| | `termVector` | `double[]` | no | In-process TF-IDF vector; excluded from serialisation. |
| `DocumentMetadata` | `documentId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `filename` | `String` | no | Original filename. |
| | `totalPages` | `int` | no | Page count extracted from the document. |
| | `passageCount` | `int` | no | Number of passages indexed. |
| | `uploadedAt` | `Instant` | no | When `DocumentUploaded` was emitted. |
| `Question` | `questionId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `documentId` | `String` | no | Which document this question targets. |
| | `sessionId` | `String` | no | Caller-supplied session identifier. |
| | `questionText` | `String` | no | Raw question text. |
| | `askedAt` | `Instant` | no | When the endpoint received the request. |
| `CitedPassage` | `passageId` | `String` | no | MUST equal a passageId in the supplied attachment. |
| | `excerpt` | `String` | no | Verbatim short phrase from the passage (10–40 words). |
| | `pageNumber` | `int` | no | Copied from the matching `PdfPassage`. |
| `CitedAnswer` | `answerable` | `boolean` | no | `true` if the document addresses the question. |
| | `answerText` | `String` | no | Synthesised answer, or `"I cannot find this in the document"`. |
| | `citations` | `List<CitedPassage>` | no | Empty list when `answerable == false`. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `ChatExchange` | `questionId` | `String` | no | — |
| | `question` | `Optional<Question>` | yes | Populated after `QuestionAsked`. |
| | `retrievedPassages` | `Optional<List<PdfPassage>>` | yes | Populated after `PassagesRetrieved`. |
| | `answer` | `Optional<CitedAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `status` | `ExchangeStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuestionAsked` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `PdfDocument` | `documentId` | `String` | no | — |
| | `metadata` | `Optional<DocumentMetadata>` | yes | Populated after `DocumentIndexed`. |
| | `passages` | `Optional<List<PdfPassage>>` | yes | Populated after `DocumentIndexed`. |
| | `status` | `DocumentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `DocumentUploaded` emitted. |

Every nullable field on `ChatExchange` and `PdfDocument` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ExchangeStatus`: `RETRIEVING`, `ANSWERING`, `ANSWERED`, `UNANSWERABLE`, `FAILED`.

`DocumentStatus`: `UPLOADING`, `INDEXED`, `FAILED`.

## Events (`PdfDocumentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DocumentUploaded` | `filename`, `pdfBase64` | → UPLOADING |
| `DocumentIndexed` | `metadata`, `passages` | → INDEXED |
| `DocumentIndexFailed` | `reason: String` | → FAILED |

## Events (`ChatSessionWorkflow` — written to `ChatSessionView`)

| Event | Payload | Exchange transition |
|---|---|---|
| `QuestionAsked` | `question` | → RETRIEVING |
| `PassagesRetrieved` | `passages: List<PdfPassage>` | → ANSWERING |
| `AnswerRecorded` | `answer: CitedAnswer` | → ANSWERED or UNANSWERABLE (based on `answerable`) |
| `ExchangeFailed` | `reason: String` | → FAILED |

`emptyState()` returns `PdfDocument.initial("")` with all `Optional` fields as `Optional.empty()` and `status = UPLOADING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ChatExchangeRow` mirrors `ChatExchange`. The full `PdfPassage.text` content is included in the view row (it is not a privacy-sensitive field, unlike the raw document in `docreview`). The `PdfPassage.termVector` field is excluded — it is `null` on all serialised forms.

The view declares ONE query for exchanges per document: `getAllExchangesForDocument(documentId): SELECT * AS exchanges FROM chat_session_view WHERE document_id = :documentId`. Ordered by `createdAt DESC`. No `WHERE status` filter on the status column (Lesson 2); the caller filters client-side.

## Task definition (`PdfChatTasks.java`)

```java
public final class PdfChatTasks {
  public static final Task<CitedAnswer> ANSWER_QUESTION = Task
      .name("Answer question from PDF")
      .description("Read the supplied passages and produce a CitedAnswer for the user question")
      .resultConformsTo(CitedAnswer.class);

  private PdfChatTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
