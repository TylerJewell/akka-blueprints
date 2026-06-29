# PLAN — wp-autotagger

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef stub fill:#131313,stroke:#888,color:#888;

  API[PostEndpoint]:::ep
  Entity[PostEntity]:::ese
  Fetcher[WpApiConsumer]:::cons
  WF[TaggingWorkflow]:::wf
  Agent[TagProposerAgent]:::agent
  Guard[TagProposalGuardrail]:::guard
  Stub[WpApiStub]:::stub
  View[PostView]:::view
  App[AppEndpoint]:::ep

  API -->|ingest| Entity
  Entity -.->|PostIngested| Fetcher
  Fetcher -->|attachBody| Entity
  Fetcher -->|start workflow| WF
  WF -->|awaitBodyStep poll| Entity
  WF -->|tagStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|applyTags tool call| Stub
  Stub -->|applyTags| Entity
  Agent -->|TagProposal| WF
  WF -->|applyTags| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as PostEndpoint
  participant E as PostEntity
  participant F as WpApiConsumer
  participant W as TaggingWorkflow
  participant A as TagProposerAgent
  participant G as TagProposalGuardrail
  participant S as WpApiStub

  U->>API: POST /api/posts
  API->>E: ingest(request)
  E-->>API: { jobId }
  E-.->>F: PostIngested
  F->>S: fetchPost(postUrl)
  S-->>F: PostBody
  F->>E: attachBody
  F->>W: start(jobId)
  W->>E: poll getPost
  E-->>W: body.isPresent()
  W->>E: markTagging
  W->>A: runSingleTask(config + attachment)
  A->>G: before-tool-call(applyTags candidate)
  G-->>A: accept
  A->>S: applyTags(postId, tags)
  S->>E: applyTags(proposal)
  A-->>W: TagProposal
  W->>E: applyTags(proposal)
  E-.->>U: SSE event(TAGS_APPLIED)
```

## State machine — `PostEntity`

```mermaid
stateDiagram-v2
  [*] --> INGESTED
  INGESTED --> BODY_FETCHED: PostBodyFetched
  BODY_FETCHED --> TAGGING: TaggingStarted
  TAGGING --> TAGS_APPLIED: TagsApplied
  INGESTED --> FAILED: PostFailed (fetch error)
  TAGGING --> FAILED: PostFailed (agent error)
  TAGS_APPLIED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PostEntity ||--o{ PostIngested : emits
  PostEntity ||--o{ PostBodyFetched : emits
  PostEntity ||--o{ TaggingStarted : emits
  PostEntity ||--o{ TagsApplied : emits
  PostEntity ||--o{ PostFailed : emits
  PostView }o--|| PostEntity : projects
  WpApiConsumer }o--|| PostEntity : subscribes
  TaggingWorkflow }o--|| PostEntity : reads-and-writes
  TagProposerAgent ||--o{ TagProposal : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PostEndpoint` | `api/PostEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PostEntity` | `application/PostEntity.java` (state in `domain/Post.java`, events in `domain/PostEvent.java`) |
| `WpApiConsumer` | `application/WpApiConsumer.java` |
| `WpApiStub` | `application/WpApiStub.java` |
| `TaggingWorkflow` | `application/TaggingWorkflow.java` |
| `TagProposerAgent` | `application/TagProposerAgent.java` (tasks in `application/TaggingTasks.java`) |
| `TagProposalGuardrail` | `application/TagProposalGuardrail.java` |
| `PostView` | `application/PostView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitBodyStep` 15 s, `tagStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(TaggingWorkflow::error)`. The 60 s on `tagStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"tagging-" + jobId` as the workflow id; the `WpApiConsumer` Consumer is allowed to redeliver `PostIngested` events because `PostEntity.attachBody` is event-version-guarded — a second fetch attempt against an already-fetched post is a no-op.
- **One agent per job**: the AutonomousAgent instance id is `"tagger-" + jobId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `TagProposalGuardrail` rejects an `applyTags` tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `tagStep` fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
