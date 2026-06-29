# PLAN — hitl-story-crafting

Architectural sketch for HITL Story Crafting. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[StoryEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[StoryWorkflow]:::wf
  SDA[StoryDrafterAgent]:::agent
  CGA[ContentGuardAgent]:::agent
  SE[StoryEntity]:::ese
  SV[StoriesView]:::view

  EP -->|start story| WF
  WF -->|draft chapter task| SDA
  WF -->|screen draft task| CGA
  SDA -->|chapterAdded| SE
  CGA -->|storyScreenedOut| SE
  EP -->|continue / end| SE
  WF -->|poll status| SE
  SE -.->|events| SV
  EP -->|getAllStories / SSE| SV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Reader
  participant EP as StoryEndpoint
  participant WF as StoryWorkflow
  participant SDA as StoryDrafterAgent
  participant CGA as ContentGuardAgent
  participant SE as StoryEntity

  Reader->>EP: POST /api/stories {premise}
  EP->>WF: start(storyId, premise)
  WF->>SDA: runSingleTask(DRAFT_CHAPTER)
  SDA-->>WF: ChapterDraft{title, body}
  WF->>CGA: runSingleTask(SCREEN_DRAFT)
  CGA-->>WF: GuardResult{passed: true}
  WF->>SE: chapterAdded -> AWAITING_DIRECTION
  Note over WF,SE: await-direction step polls every 5s
  Reader->>EP: POST /api/stories/{id}/continue {direction}
  EP->>SE: directionProvided -> DRAFTING
  WF->>SE: getStory -> DRAFTING
  WF->>SDA: runSingleTask(DRAFT_CHAPTER) [next chapter]
  SDA-->>WF: ChapterDraft{title, body}
  WF->>CGA: runSingleTask(SCREEN_DRAFT)
  CGA-->>WF: GuardResult{passed: true}
  WF->>SE: chapterAdded -> AWAITING_DIRECTION
  Reader->>EP: POST /api/stories/{id}/end
  EP->>SE: storyCompleted -> COMPLETED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> AWAITING_DIRECTION: StoryStarted + ChapterAdded
  AWAITING_DIRECTION --> DRAFTING: DirectionProvided
  DRAFTING --> AWAITING_DIRECTION: ChapterAdded
  AWAITING_DIRECTION --> COMPLETED: StoryCompleted
  DRAFTING --> SCREENED_OUT: StoryScreenedOut
  AWAITING_DIRECTION --> SCREENED_OUT: StoryScreenedOut
  SCREENED_OUT --> [*]
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  STORY ||--o{ STORY_EVENT : emits
  STORY {
    string id
    string premise
    string status
    int turnCount
    string startedAt
    string lastChapterAt
    string completedAt
    string screenOutReason
  }
  CHAPTER {
    int turnNumber
    string title
    string body
    string direction
  }
  STORY_EVENT {
    string type
    string occurredAt
  }
  STORIES_VIEW {
    string id
    string status
    int turnCount
  }
  STORY ||--o{ CHAPTER : contains
  STORY ||--|| STORIES_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| StoryDrafterAgent | `application/StoryDrafterAgent.java` |
| ContentGuardAgent | `application/ContentGuardAgent.java` |
| StoryWorkflow | `application/StoryWorkflow.java` |
| StoryTasks | `application/StoryTasks.java` |
| StoryEntity | `application/StoryEntity.java` |
| StoriesView | `application/StoriesView.java` |
| StoryEndpoint | `api/StoryEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Story / Chapter / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `draftChapterStep` and `screenDraftStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would expire before a language model responds (Lesson 4).
- **Await-direction step.** The workflow does not block a thread; `awaitDirectionStep` reads `StoryEntity.getStory`, and on `AWAITING_DIRECTION` self-schedules a 5-second resume timer until the reader acts.
- **Loop termination.** On `COMPLETED` or `SCREENED_OUT`, the await-direction step returns a terminal workflow result, ending the workflow without external cancellation.
- **Idempotency.** `storyId` is the workflow id and the entity id; re-delivery of `chapterAdded` is absorbed by event-applier checks on `turnCount`.
- **Content guard placement.** `screenDraftStep` runs before `chapterAdded` is written. A rejected draft never touches `StoryEntity`, so the state machine never enters an inconsistent partially-written chapter state.
