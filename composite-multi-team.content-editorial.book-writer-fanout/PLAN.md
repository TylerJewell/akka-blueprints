# PLAN — book-writer-fanout

Architectural sketch. All four mermaid diagrams render on the Architecture tab with the Lesson-24 theme variables and CSS overrides for state and edge labels.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  SIM[RequestSimulator]:::ta -.->|every 45s| RQ[BookRequestQueue]:::ese
  EP[BookEndpoint]:::ep --> RQ
  RQ -. BookRequestQueued .-> RC[BookRequestConsumer]:::cons
  RC --> WF[BookWritingWorkflow]:::wf
  WF --> OA[OutlineAgent]:::agent
  WF --> CA[ChapterAgent]:::agent
  CA --> WS[WebSearchEndpoint]:::ep
  WF --> BE[BookEntity]:::ese
  WF --> CE[ChapterEntity]:::ese
  CE -. ChapterDrafted .-> EV[ChapterEvalConsumer]:::cons
  EV --> CE
  BE -. events .-> V[BooksView]:::view
  CE -. events .-> V
  V --> EP
  APP[AppEndpoint]:::ep
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor User
  User->>BookEndpoint: POST /api/books {topic}
  BookEndpoint->>BookRequestQueue: enqueueRequest(topic)
  BookRequestQueue-->>BookRequestConsumer: BookRequestQueued
  BookRequestConsumer->>BookWritingWorkflow: start(bookId, topic)
  BookWritingWorkflow->>OutlineAgent: runSingleTask(OUTLINE)
  OutlineAgent-->>BookWritingWorkflow: BookOutline
  BookWritingWorkflow->>BookEntity: recordOutline
  Note over BookWritingWorkflow: create one ChapterEntity per chapter
  loop each chapter
    BookWritingWorkflow->>ChapterAgent: runSingleTask(WRITE_CHAPTER)
    ChapterAgent->>WebSearchEndpoint: POST /api/search (guarded)
    WebSearchEndpoint-->>ChapterAgent: results or 403
    ChapterAgent-->>BookWritingWorkflow: ChapterDraft
    BookWritingWorkflow->>ChapterEntity: recordDraft
    ChapterEntity-->>ChapterEvalConsumer: ChapterDrafted
    ChapterEvalConsumer->>ChapterEntity: recordEvaluation(score)
  end
  Note over BookWritingWorkflow: documentation gate — every chapter present and non-empty
  BookWritingWorkflow->>BookEntity: recordConsolidation, complete
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> OUTLINED: OutlineRecorded
  OUTLINED --> WRITING: first ChapterProgressed
  WRITING --> WRITING: ChapterProgressed
  WRITING --> CONSOLIDATING: all chapters drafted
  CONSOLIDATING --> COMPLETED: gate passes
  CONSOLIDATING --> FAILED: gate fails
  WRITING --> FAILED: step retries exhausted
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BOOK ||--o{ CHAPTER : contains
  BOOK {
    string id
    string topic
    string status
    string title
    string manuscript
  }
  CHAPTER {
    string chapterId
    string bookId
    int index
    string title
    string status
    double qualityScore
  }
  BOOK_REQUEST_QUEUE {
    string topic
  }
  BOOKS_VIEW {
    string id
    string status
    json chapters
  }
  BOOK ||--|| BOOKS_VIEW : projects
  CHAPTER ||--|| BOOKS_VIEW : projects
```

## Component table

| Component | Kind | Path (generated) |
|---|---|---|
| OutlineAgent | AutonomousAgent | `application/OutlineAgent.java` |
| ChapterAgent | AutonomousAgent | `application/ChapterAgent.java` |
| BookWritingTasks | task definitions | `application/BookWritingTasks.java` |
| BookWritingWorkflow | Workflow | `application/BookWritingWorkflow.java` |
| BookEntity | EventSourcedEntity | `application/BookEntity.java` |
| ChapterEntity | EventSourcedEntity | `application/ChapterEntity.java` |
| BookRequestQueue | EventSourcedEntity | `application/BookRequestQueue.java` |
| BooksView | View | `application/BooksView.java` |
| BookRequestConsumer | Consumer | `application/BookRequestConsumer.java` |
| ChapterEvalConsumer | Consumer | `application/ChapterEvalConsumer.java` |
| RequestSimulator | TimedAction | `application/RequestSimulator.java` |
| WebSearchEndpoint | HttpEndpoint | `api/WebSearchEndpoint.java` |
| BookEndpoint | HttpEndpoint | `api/BookEndpoint.java` |
| AppEndpoint | HttpEndpoint | `api/AppEndpoint.java` |
| Bootstrap | service-setup | `Bootstrap.java` |
| Book, Chapter, BookStatus, ChapterStatus, events | domain | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `outlineStep`, `writeChaptersStep`, and `consolidateStep` each call agents; set `stepTimeout(120s)` per step (Lesson 4). `writeChaptersStep` drafts chapters sequentially, so its budget covers all chapters in one book — for the canned 3–5 chapter outlines this stays within 120s; for larger outlines split into one step per chapter keyed by index.
- **Idempotency.** ChapterEntity ids are deterministic (`chapter-{bookId}-{index}`), so a workflow retry re-targets the same chapter rather than creating duplicates. `recordDraft` is a no-op when the chapter is already `DRAFTED`.
- **Eval is non-blocking.** ChapterEvalConsumer scores out of band; `consolidateStep` reads whatever score is present and treats a missing score as unscored, never blocking on it. The blocking gate is completeness (every chapter has non-empty markdown), realised in `consolidateStep`.
- **Compensation.** A failed completeness gate records `BookFailed` with a reason rather than leaving the book mid-flight; `defaultStepRecovery(maxRetries(2).failoverTo(error))` routes exhausted retries to the same terminal failure.
