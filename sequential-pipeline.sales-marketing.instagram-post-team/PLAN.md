# PLAN — instagram-post-team

Architectural sketch. All four mermaid diagrams + the component table.

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

  SIM[BriefSimulator]:::ta -. drips .-> BQ[BriefQueue]:::ese
  EP[PostEndpoint]:::ep --> BQ
  BQ -. events .-> CON[BriefConsumer]:::cons
  CON --> WF[PostWorkflow]:::wf
  WF --> CA[CaptionAgent]:::agent
  WF --> IA[ImagePromptAgent]:::agent
  WF --> PE[PostEntity]:::ese
  PE -. events .-> PV[PostsView]:::view
  EP --> PE
  EP --> PV
  APP[AppEndpoint]:::ep
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor M as Marketer
  participant EP as PostEndpoint
  participant BQ as BriefQueue
  participant CON as BriefConsumer
  participant WF as PostWorkflow
  participant CA as CaptionAgent
  participant IA as ImagePromptAgent
  participant PE as PostEntity
  M->>EP: POST /api/briefs
  EP->>BQ: enqueue(brief)
  BQ-->>CON: BriefQueued
  CON->>WF: start(postId)
  WF->>CA: write(brief)
  CA-->>WF: CaptionDraft
  Note over WF: before-agent-response guardrail checks the caption
  WF->>PE: recordCaption (pass)
  WF->>IA: draw(brief, caption)
  IA-->>WF: ImagePrompt
  Note over WF: before-agent-response guardrail checks the image prompt
  WF->>PE: recordImagePrompt (pass)
  WF->>PE: markReady
  Note over PE: status READY
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> COMPOSING
  COMPOSING --> PROMPTING: CaptionWritten (check pass)
  COMPOSING --> BLOCKED: PostBlocked (check fail)
  PROMPTING --> READY: ImagePromptWritten + PostReady (check pass)
  PROMPTING --> BLOCKED: PostBlocked (check fail)
  COMPOSING --> FAILED: retries exhausted
  PROMPTING --> FAILED: retries exhausted
  READY --> [*]
  BLOCKED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BRIEF_QUEUE ||--o{ POST_ENTITY : starts
  POST_ENTITY ||--|| POSTS_VIEW : projects
  BRIEF_QUEUE {
    string id
    string brief
  }
  POST_ENTITY {
    string id
    string brief
    enum status
    string caption
    string imagePrompt
    boolean safetyCheckPassed
  }
  POSTS_VIEW {
    string id
    enum status
    string caption
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| CaptionAgent | `application/CaptionAgent.java` |
| ImagePromptAgent | `application/ImagePromptAgent.java` |
| PostWorkflow | `application/PostWorkflow.java` |
| PostEntity | `domain/PostEntity.java` |
| BriefQueue | `domain/BriefQueue.java` |
| PostsView | `application/PostsView.java` |
| BriefConsumer | `application/BriefConsumer.java` |
| BriefSimulator | `application/BriefSimulator.java` |
| PostEndpoint | `api/PostEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- Workflow step timeouts: 60s on `captionStep` and `imagePromptStep` (agent calls take 10–30s — the 5s default would retry forever, Lesson 4). `finalizeStep` is a local entity write and needs no override.
- Idempotency: the workflow id is the post id; re-delivery of `BriefQueued` reuses the same workflow instance. Entity commands are guarded by current status, so a duplicate `markReady` after `READY` is a no-op.
- Compensation: a failed before-agent-response check transitions the post to `BLOCKED` with the check notes as the block reason — no further step runs. `defaultStepRecovery(maxRetries(2).failoverTo(error))` ends the workflow on exhausted retries; the post stays at its last recorded status (`FAILED` on the error path) for operator inspection.
