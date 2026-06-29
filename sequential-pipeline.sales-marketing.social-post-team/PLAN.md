# PLAN — social-post-team

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

  SIM[ConceptSimulator]:::ta -. drips .-> CQ[ConceptQueue]:::ese
  EP[PostEndpoint]:::ep --> CQ
  CQ -. events .-> CON[ConceptConsumer]:::cons
  CON --> WF[PostWorkflow]:::wf
  WF --> RA[ResearchAgent]:::agent
  WF --> CA[CopyAgent]:::agent
  WF --> VA[VisualDirectorAgent]:::agent
  RA --> WT[WebTools]:::ep
  WF --> PE[PostEntity]:::ese
  PE -. events .-> PV[PostsView]:::view
  EP --> PE
  EP --> PV
  MON[StalePostMonitor]:::ta -. ticks .-> PV
  MON --> PE
  APP[AppEndpoint]:::ep
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor M as Marketer
  participant EP as PostEndpoint
  participant CQ as ConceptQueue
  participant CON as ConceptConsumer
  participant WF as PostWorkflow
  participant RA as ResearchAgent
  participant CA as CopyAgent
  participant VA as VisualDirectorAgent
  participant PE as PostEntity
  M->>EP: POST /api/concepts
  EP->>CQ: enqueue(concept)
  CQ-->>CON: ConceptQueued
  CON->>WF: start(postId)
  WF->>RA: research(concept)
  RA-->>WF: ResearchNotes
  WF->>PE: recordResearch
  WF->>CA: compose(input)
  WF->>VA: direct(input)
  WF->>PE: recordCompose (brand check passes)
  Note over WF,PE: status AWAITING_APPROVAL — workflow polls on a 5s timer
  M->>EP: POST /api/posts/{id}/approve
  EP->>PE: approve
  WF->>PE: recordPublication
  Note over PE: status PUBLISHED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RESEARCHING
  RESEARCHING --> COMPOSING: PostResearched
  COMPOSING --> AWAITING_APPROVAL: PostComposed + BrandChecked(pass)
  COMPOSING --> REJECTED: BrandChecked(fail)
  AWAITING_APPROVAL --> APPROVED: PostApproved
  AWAITING_APPROVAL --> REJECTED: PostRejected
  AWAITING_APPROVAL --> ESCALATED: PostEscalated
  APPROVED --> PUBLISHED: PostPublished
  REJECTED --> [*]
  ESCALATED --> [*]
  PUBLISHED --> [*]
```

## Entity model

```mermaid
erDiagram
  CONCEPT_QUEUE ||--o{ POST_ENTITY : starts
  POST_ENTITY ||--|| POSTS_VIEW : projects
  CONCEPT_QUEUE {
    string id
    string concept
  }
  POST_ENTITY {
    string id
    string concept
    enum status
    string caption
    string visualBrief
    string publishedPermalink
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
| ResearchAgent | `application/ResearchAgent.java` |
| CopyAgent | `application/CopyAgent.java` |
| VisualDirectorAgent | `application/VisualDirectorAgent.java` |
| PostWorkflow | `application/PostWorkflow.java` |
| PostEntity | `domain/PostEntity.java` |
| ConceptQueue | `domain/ConceptQueue.java` |
| PostsView | `application/PostsView.java` |
| ConceptConsumer | `application/ConceptConsumer.java` |
| ConceptSimulator | `application/ConceptSimulator.java` |
| StalePostMonitor | `application/StalePostMonitor.java` |
| WebTools | `api/WebTools.java` |
| PostEndpoint | `api/PostEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- Workflow step timeouts: 60s on `researchStep`, `composeStep`, `brandCheckStep`, `publishStep` (agent calls take 10–30s — the 5s default would retry forever, Lesson 4). `awaitApprovalStep` self-schedules a 5s resume timer and re-polls `PostEntity.getPost`.
- Idempotency: workflow id is the post id; re-delivery of `ConceptQueued` reuses the same workflow instance. Entity commands are guarded by current status so a duplicate `approve` after `PUBLISHED` is a no-op.
- Compensation: `brandCheckStep` failure transitions the post to `REJECTED` with the check notes as reason — no publish side effect runs. `defaultStepRecovery(maxRetries(2).failoverTo(error))` ends the workflow on exhausted retries; the post stays at its last recorded status for operator inspection.
