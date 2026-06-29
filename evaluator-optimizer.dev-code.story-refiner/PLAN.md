# PLAN — story-refiner

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

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

  Drafter[StoryDrafterAgent]:::agent
  Reviewer[ReviewerAgent]:::agent

  WF[RefinementWorkflow]:::wf
  Story[StoryEntity]:::ese
  Queue[RequestQueue]:::ese
  View[StoriesView]:::view
  Consumer[StoryRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[StoryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue brief| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|draft / revise| Drafter
  WF -->|review| Reviewer
  WF -->|emit events| Story
  Story -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Story
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as StoryEndpoint
  participant Q as RequestQueue
  participant C as StoryRequestConsumer
  participant W as RefinementWorkflow
  participant D as StoryDrafterAgent
  participant R as ReviewerAgent
  participant E as StoryEntity
  participant V as StoriesView

  U->>API: POST /api/stories {featureDescription, targetTeam}
  API->>Q: append BriefSubmitted
  API-->>U: 202 {storyId}
  Q->>C: BriefSubmitted
  C->>W: start({storyId, featureDescription, targetTeam, maxAttempts=4})
  W->>E: emit StoryCreated (DRAFTING)

  W->>D: DRAFT(featureDescription, targetTeam)
  D-->>W: StoryDraft #1 (role, goal, benefit, 3 ACs)
  W->>E: emit AttemptDrafted (n=1)
  W->>E: status REVIEWING
  W->>R: REVIEW(StoryDraft #1)
  R-->>W: Review{REVISE, score=3, 3 bullets}
  W->>E: emit AttemptReviewed (n=1, REVISE)

  W->>D: REVISE_DRAFT(featureDescription, prior, notes)
  D-->>W: StoryDraft #2 (tightened ACs, clearer goal)
  W->>E: emit AttemptDrafted (n=2)
  W->>E: status REVIEWING
  W->>R: REVIEW(StoryDraft #2)
  R-->>W: Review{APPROVE, score=5, rationale}
  W->>E: emit AttemptReviewed (n=2, APPROVE)
  W->>E: emit StoryApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `StoryEntity`

```mermaid
stateDiagram-v2
  [*] --> DRAFTING
  DRAFTING --> REVIEWING: AttemptDrafted
  REVIEWING --> DRAFTING: Review = REVISE, attempts < max
  REVIEWING --> APPROVED: Review = APPROVE
  REVIEWING --> REJECTED_FINAL: Review = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  StoryEntity ||--o{ StoryCreated : emits
  StoryEntity ||--o{ AttemptDrafted : emits
  StoryEntity ||--o{ AttemptReviewed : emits
  StoryEntity ||--o{ StoryApproved : emits
  StoryEntity ||--o{ StoryRejectedFinal : emits
  StoryEntity ||--o{ EvalRecorded : emits
  StoriesView }o--|| StoryEntity : projects
  RequestQueue ||--o{ BriefSubmitted : emits
  StoryRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `StoryDrafterAgent` | `application/StoryDrafterAgent.java` |
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `StoryTasks` | `application/StoryTasks.java` |
| `RefinementWorkflow` | `application/RefinementWorkflow.java` |
| `StoryEntity` | `application/StoryEntity.java` (state in `domain/Story.java`, events in `domain/StoryEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `StoriesView` | `application/StoriesView.java` |
| `StoryRequestConsumer` | `application/StoryRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `StoryEndpoint` | `api/StoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `draftStep` and `reviewStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `StoryEndpoint.submit` uses `(featureDescription, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(storyId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `story-refiner.refinement.max-attempts` (default 4). The workflow checks the count BEFORE calling `draftStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt on ceiling exhaustion preserves the best draft and every review on the entity.
- **No guardrail step:** unlike content-generation loops, story refinement has no deterministic format ceiling to enforce before the reviewer runs. The reviewer's rubric covers all quality dimensions; adding a separate pre-review guardrail is an extension point, not a baseline requirement.
