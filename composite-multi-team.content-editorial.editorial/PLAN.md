# PLAN — editorial-desk

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab with the Akka theme variables and the Lesson 24 state-label CSS overrides.

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

  Chief[EditorInChief]:::agent
  RLead[ResearchLead]:::agent
  Rsch[Researcher]:::agent
  WLead[WritingLead]:::agent
  Writer[Writer]:::agent
  Rev[Reviewer]:::agent

  Edit[EditorialWorkflow]:::wf
  WriterWF[WriterWorkflow]:::wf

  Doc[DocumentEntity]:::ese
  Sect[SectionEntity]:::ese
  Queue[StoryQueue]:::ese
  DBoard[DocumentBoardView]:::view
  SBoard[SectionBoardView]:::view
  StoryC[StoryRequestConsumer]:::cons
  EvalC[StageEvalConsumer]:::cons
  Sim[StorySimulator]:::ta
  Stuck[StuckSectionMonitor]:::ta
  API[EditorialEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit story| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| StoryC
  StoryC -->|create| Doc
  StoryC -->|start workflow| Edit

  Edit -->|ASSIGN / FINALIZE| Chief
  Edit -->|PLAN_RESEARCH / SYNTHESIZE| RLead
  Edit -->|RESEARCH x N| Rsch
  Edit -->|PLAN_SECTIONS| WLead
  Edit -->|REVIEW x axes| Rev
  Edit -->|create one per section| Sect
  Edit -->|record stages| Doc
  Edit -->|poll sections| SBoard

  WriterWF -->|poll board| SBoard
  WriterWF -->|atomic claim| Sect
  WriterWF -->|WRITE_SECTION| Writer

  Rsch -.->|DocumentTools write| Doc
  Writer -.->|DocumentTools write| Sect
  Rev -.->|DocumentTools write| Doc

  Doc -.->|projects| DBoard
  Sect -.->|projects| SBoard
  Doc -.->|stage events| EvalC
  EvalC -->|record eval| Doc
  Stuck -.->|every 30s| Sect

  API -->|query / SSE| DBoard
  API -->|query / SSE| SBoard
  API -->|compliance review| Doc
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions, scheduled ticks, and guarded tool writes. `Researcher`, `Writer`, and `Reviewer` are each one agent class run as several instances — researcher instances per subtopic, `writer-1`/`writer-2`, and one reviewer per axis (`factcheck`, `style`, `legal`). The top-level `EditorialWorkflow` is the editor-in-chief; the three desks each run a different internal coordination capability: research delegates and fans in, writing is a team over the shared `SectionBoardView`, review is a moderated panel feeding `ModerationRule`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as EditorialEndpoint
  participant Q as StoryQueue
  participant C as StoryRequestConsumer
  participant EW as EditorialWorkflow
  participant CH as EditorInChief
  participant RL as ResearchLead
  participant RS as Researcher
  participant S as SectionEntity
  participant WW as WriterWorkflow
  participant W as Writer
  participant RV as Reviewer
  participant D as DocumentEntity

  U->>API: POST /api/stories {topic}
  API->>Q: submitStory(brief)
  API-->>U: 202 {storyId}
  Q->>C: StorySubmitted
  C->>D: createDocument
  C->>EW: start({storyId})
  EW->>CH: ASSIGN(topic)
  CH-->>EW: EditorialBrief
  EW->>D: assignBrief
  EW->>RL: PLAN_RESEARCH
  RL-->>EW: ResearchPlan{subtopics}
  EW->>RS: RESEARCH(subtopic) x N
  RS-->>D: appendResearchNote (guarded)
  EW->>RL: SYNTHESIZE(notes)
  RL-->>EW: ResearchDigest
  EW->>D: recordDigest
  EW->>D: stage event -> StageEvalConsumer records eval
  EW->>WLead: PLAN_SECTIONS
  Note over EW: writes one SectionEntity per section (OPEN)
  EW->>S: createSection x M
  Note over WW: writer loops already polling the board
  WW->>S: claim(writer-1) (atomic, single winner)
  WW->>W: WRITE_SECTION(section)
  W-->>S: writeSection content (guarded)
  WW->>S: recordWritten
  Note over EW: poll until all sections WRITTEN
  EW->>D: assembleDraft
  EW->>RV: REVIEW(axis, draft) x 3
  RV-->>D: appendReviewNote (guarded)
  EW->>EW: ModerationRule(notes) -> PASS
  EW->>D: recordReview(PASS)
  EW->>CH: FINALIZE(draft)
  Note over CH: before-agent-response guardrail vets the Article
  CH-->>EW: Article
  EW->>D: publish(article, url)
  D-->>API: SSE status=PUBLISHED
```

## State machine — `DocumentEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED: createDocument
  SUBMITTED --> ASSIGNED: assignBrief
  ASSIGNED --> RESEARCHING: research begins
  RESEARCHING --> RESEARCHED: recordDigest
  RESEARCHED --> WRITING: recordSections
  WRITING --> DRAFTED: assembleDraft (all sections written)
  DRAFTED --> REVIEWING: panel runs
  REVIEWING --> APPROVED: recordReview PASS
  REVIEWING --> WRITING: requestRevision (one bounded round)
  APPROVED --> PUBLISHED: publish (output guardrail passes)
  APPROVED --> APPROVED: recordPublishBlock (guardrail refusal)
  PUBLISHED --> PUBLISHED: recordComplianceReview (post-publication, non-blocking)
  PUBLISHED --> [*]
```

## Entity model

```mermaid
erDiagram
  DocumentEntity ||--o{ DocumentCreated : emits
  DocumentEntity ||--o{ BriefAssigned : emits
  DocumentEntity ||--o{ ResearchSynthesized : emits
  DocumentEntity ||--o{ SectionsPlanned : emits
  DocumentEntity ||--o{ DraftAssembled : emits
  DocumentEntity ||--o{ ReviewCompleted : emits
  DocumentEntity ||--o{ RevisionRequested : emits
  DocumentEntity ||--o{ ArticlePublished : emits
  DocumentEntity ||--o{ StageEvaluated : emits
  DocumentEntity ||--o{ ComplianceReviewRecorded : emits
  DocumentBoardView }o--|| DocumentEntity : projects
  StageEvalConsumer }o--|| DocumentEntity : subscribes
  DocumentEntity ||--o{ SectionEntity : "owns M sections"
  SectionEntity ||--o{ SectionCreated : emits
  SectionEntity ||--o{ SectionClaimed : emits
  SectionEntity ||--o{ SectionWritten : emits
  SectionEntity ||--o{ SectionReleased : emits
  SectionBoardView }o--|| SectionEntity : projects
  StoryQueue ||--o{ StorySubmitted : emits
  StoryRequestConsumer }o--|| StoryQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `EditorInChief` | `application/EditorInChief.java` |
| `ResearchLead` | `application/ResearchLead.java` |
| `Researcher` | `application/Researcher.java` |
| `WritingLead` | `application/WritingLead.java` |
| `Writer` | `application/Writer.java` |
| `Reviewer` | `application/Reviewer.java` |
| `EditorialTasks` | `application/EditorialTasks.java` |
| `DocumentTools` | `application/DocumentTools.java` |
| `ModerationRule` | `application/ModerationRule.java` |
| `StageEvaluator` | `application/StageEvaluator.java` |
| `EditorialWorkflow` | `application/EditorialWorkflow.java` |
| `WriterWorkflow` | `application/WriterWorkflow.java` |
| `DocumentEntity` | `application/DocumentEntity.java` (state in `domain/Document.java`, events in `domain/DocumentEvent.java`) |
| `SectionEntity` | `application/SectionEntity.java` (state in `domain/Section.java`, events in `domain/SectionEvent.java`) |
| `StoryQueue` | `application/StoryQueue.java` |
| `DocumentBoardView` | `application/DocumentBoardView.java` |
| `SectionBoardView` | `application/SectionBoardView.java` |
| `StoryRequestConsumer` | `application/StoryRequestConsumer.java` |
| `StageEvalConsumer` | `application/StageEvalConsumer.java` |
| `StorySimulator` | `application/StorySimulator.java` |
| `StuckSectionMonitor` | `application/StuckSectionMonitor.java` |
| `EditorialEndpoint` | `api/EditorialEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **6 autonomous-agent · 2 workflow · 3 event-sourced-entity · 2 view · 2 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Two coordination primitives sit under one pipeline.** The top-level `EditorialWorkflow` is a sequential delegation: each stage runs and writes its result onto the shared `DocumentEntity` before the next begins. The writing stage hands off to an independent team: the workflow seeds the board and then waits, while the per-writer `WriterWorkflow` loops claim and fill sections on their own clock.
- **Atomic claim is the writing-team primitive.** `SectionEntity` is a single-writer; `claim(writerId)` emits `SectionClaimed` only when the current status is `OPEN`. Two writer workflows that read the same `OPEN` section from the board and both call `claim` are serialised by the entity — the first wins, the second receives the already-claimed `Section` and returns to polling. No lock, no external queue.
- **The writing wait is a poll, not a block.** `EditorialWorkflow.writingStep` queries `SectionBoardView` for this story's sections; if any are not `WRITTEN`, it self-schedules a 5 s resume timer and pauses. An idle writing stage is a paused workflow, not a busy loop.
- **Workflow step timeouts:** every step that calls an agent sets an explicit `stepTimeout` (Lesson 4) — `briefStep` 60 s, `researchStep` 120 s (it fans out several researcher calls), `reviewStep` 90 s, `publishStep` 60 s, and `WriterWorkflow.writeStep` 90 s. The default 5 s timeout would expire mid-LLM-call.
- **Bounded revision loop.** A `REVISE` verdict resets the named sections to `OPEN` and returns the document to `WRITING`, but only once (`revisionCount < 1`); a second `REVISE` accepts the draft and proceeds to publish, so the pipeline always terminates.
- **The output guardrail can stall, not crash.** If the G1 before-agent-response guardrail refuses the final article, `publishStep` records the block and ends with the document left `APPROVED`; nothing is published and the reason is visible in the UI.
- **Release for liveness:** `StuckSectionMonitor` returns a section claimed-but-idle for more than two minutes to `OPEN`, so a writer that fails mid-section does not strand the board. `release` is a no-op unless the section is `CLAIMED`.
- **The stage eval is downstream and non-blocking.** `StageEvalConsumer` subscribes to `DocumentEntity` events and records a `StageEval` after a stage result lands; it never gates the pipeline (control E1).
- **Compliance review is on the loop.** `recordComplianceReview` is accepted only when the document is `PUBLISHED` and never changes that status (control HO1).
- **Idempotency:** deterministic `sectionId = storyId + "-s" + index` makes `createSection` idempotent if `writingStep` is retried; `storyId` is the `EditorialWorkflow` id so a redelivered `StorySubmitted` starts the same workflow, not a duplicate.
```
