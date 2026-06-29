# Data model — book-writer-fanout

Every Java record the generated system defines. Nullable lifecycle fields are `Optional<T>`
(Lesson 6); the JSON on the wire is the raw value or `null`.

## Book (BookEntity state + BooksView row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| id | String | no | Book id (UUID) |
| topic | Optional<String> | yes | Submitted topic; empty in `emptyState()` |
| status | BookStatus | no | Lifecycle state |
| title | Optional<String> | yes | Book title from the outline |
| chapters | List<ChapterSummary> | no | Empty until outline recorded |
| chapterCount | Optional<Integer> | yes | Number of planned chapters |
| chaptersDrafted | Optional<Integer> | yes | Count drafted so far |
| manuscript | Optional<String> | yes | Consolidated markdown |
| outlinedAt | Optional<Instant> | yes | When the outline was recorded |
| consolidatedAt | Optional<Instant> | yes | When consolidation completed |
| completedAt | Optional<Instant> | yes | When the book reached COMPLETED |
| failedAt | Optional<Instant> | yes | When the book reached FAILED |
| failureReason | Optional<String> | yes | Why the book failed |

`emptyState()` returns `Book.initial("")` — no `commandContext()` reference (Lesson 3).

## Chapter (ChapterEntity state)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| chapterId | String | no | `chapter-{bookId}-{index}` |
| bookId | String | no | Owning book |
| index | int | no | 0-based order |
| title | String | no | Chapter title from the outline |
| brief | String | no | One-sentence scope |
| status | ChapterStatus | no | Lifecycle state |
| draftMarkdown | Optional<String> | yes | Drafted chapter body |
| qualityScore | Optional<Double> | yes | Eval score 0–1 |
| draftedAt | Optional<Instant> | yes | When drafted |
| evaluatedAt | Optional<Instant> | yes | When scored |

## ChapterSummary (embedded in the BooksView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| chapterId | String | no | Chapter id |
| index | int | no | Order |
| title | String | no | Chapter title |
| status | String | no | `ChapterStatus.name()` (String, not enum — avoids the view enum-index restriction, Lesson 2) |
| qualityScore | Optional<Double> | yes | Latest eval score |

## Agent result records

| Record | Fields | Used by |
|---|---|---|
| BookOutline | `String title`, `List<ChapterPlan> chapters` | OUTLINE task result |
| ChapterPlan | `String title`, `String brief` | inside BookOutline |
| ChapterDraft | `String markdown` | WRITE_CHAPTER task result |
| SearchResult | `String title`, `String snippet`, `String url` | WebSearchEndpoint response |

## Status enums

- **BookStatus:** `REQUESTED · OUTLINED · WRITING · CONSOLIDATING · COMPLETED · FAILED`
- **ChapterStatus:** `PENDING · DRAFTING · DRAFTED · APPROVED · REJECTED`

## Events

### BookEntity

| Event | Trigger |
|---|---|
| BookRequested | `requestBook` — workflow starts for a topic |
| OutlineRecorded | `recordOutline` — OutlineAgent returned a plan; sets title, chapters, chapterCount |
| ChapterProgressed | `progressChapter` — a chapter finished drafting; increments chaptersDrafted, moves to WRITING |
| ManuscriptConsolidated | `recordConsolidation` — chapters joined into a manuscript |
| BookCompleted | `complete` — completeness gate passed |
| BookFailed | `fail` — completeness gate failed or step retries exhausted |

### ChapterEntity

| Event | Trigger |
|---|---|
| ChapterDrafted | `recordDraft` — ChapterAgent returned markdown; sets draftMarkdown, draftedAt, status DRAFTED |
| ChapterEvaluated | `recordEvaluation` — ChapterEvalConsumer scored the chapter |
| ChapterRejected | `reject` — score below threshold 0.5; status REJECTED for redraft |

### BookRequestQueue

| Event | Trigger |
|---|---|
| BookRequestQueued | `enqueueRequest` — a topic submitted via endpoint or simulator |

## View row

`BooksView` row type is `Book`. The `TableUpdater` consumes both `BookEntity` and
`ChapterEntity` events; chapter events update the matching `ChapterSummary` inside the book
row. One query: `getAllBooks` → `SELECT * AS books FROM books_view` (no `WHERE status` filter
— Akka cannot auto-index enum columns, Lesson 2; filter client-side). `streamAllBooks` backs
the SSE endpoint.
