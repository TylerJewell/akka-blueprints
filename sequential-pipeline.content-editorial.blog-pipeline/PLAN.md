# PLAN — AI Blog Writer Pipeline with Ollama

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[PostEndpoint]:::ep
  Entity[PostEntity]:::ese
  WF[BlogPipelineWorkflow]:::wf
  Agent[BlogWriterAgent]:::agent
  Research[ResearchTools]:::tool
  Outline[OutlineTools]:::tool
  Draft[DraftTools]:::tool
  Edit[EditTools]:::tool
  Publish[PublishTools]:::tool
  Guard[ContentPolicyGuardrail]:::guard
  View[PostView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|researchStep runSingleTask| Agent
  WF -->|outlineStep runSingleTask| Agent
  WF -->|draftStep runSingleTask| Agent
  WF -->|editStep runSingleTask| Agent
  WF -->|publishStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|invokes| Research
  Agent -->|invokes| Outline
  Agent -->|invokes| Draft
  Agent -->|invokes| Edit
  Agent -->|invokes| Publish
  Guard -->|recordGuardrailBlock| Entity
  Guard -->|recordContentCleared| Entity
  Agent -->|ResearchNotes / Outline / Draft / EditedDraft / Post| WF
  WF -->|record each phase result| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as PostEndpoint
  participant E as PostEntity
  participant W as BlogPipelineWorkflow
  participant A as BlogWriterAgent
  participant G as ContentPolicyGuardrail
  participant T as Tools (Research/.../Publish)

  U->>API: POST /api/posts { topic, postType }
  API->>E: create(topic, postType)
  E-->>API: { postId }
  API->>W: start(postId, topic, postType)
  W->>E: startResearch
  W->>A: runSingleTask(RESEARCH_TOPIC, topic)
  A->>T: searchReferences + fetchSummary
  T-->>A: List<Reference>
  A-->>G: before-agent-response(ResearchNotes prose)
  G-->>W: accept → ContentCleared
  W->>E: recordResearch + recordContentCleared
  W->>A: runSingleTask(OUTLINE_POST, research)
  A->>T: structureSections + expandKeyPoints
  T-->>A: List<OutlineSection>
  A-->>G: before-agent-response(Outline prose)
  G-->>W: accept → ContentCleared
  W->>E: recordOutline + recordContentCleared
  W->>A: runSingleTask(DRAFT_POST, outline)
  A->>T: writeParagraph + composeTile
  T-->>A: Draft
  A-->>G: before-agent-response(Draft prose)
  G-->>W: accept → ContentCleared
  W->>E: recordDraft + recordContentCleared
  W->>A: runSingleTask(EDIT_POST, draft)
  A->>T: applyToneAdjustments + checkReadability
  T-->>A: EditedDraft
  A-->>G: before-agent-response(EditedDraft prose)
  G-->>W: accept → ContentCleared
  W->>E: recordEditedDraft + recordContentCleared
  W->>A: runSingleTask(PUBLISH_POST, editedDraft)
  A->>T: formatPost + collectReferences
  T-->>A: Post
  A-->>G: before-agent-response(Post prose)
  G-->>W: accept → ContentCleared
  W->>E: recordPost + recordContentCleared
  E-.->>U: SSE event(PUBLISHED)
```

## State machine — `PostEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RESEARCHING: ResearchStarted
  RESEARCHING --> RESEARCHED: ResearchCollected
  RESEARCHED --> OUTLINING: OutlineStarted
  OUTLINING --> OUTLINED: OutlineProduced
  OUTLINED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: DraftWritten
  DRAFTED --> EDITING: EditStarted
  EDITING --> EDITED: EditApplied
  EDITED --> PUBLISHING: PublishStarted
  PUBLISHING --> PUBLISHED: PostPublished
  RESEARCHING --> FAILED: PostFailed
  OUTLINING --> FAILED: PostFailed
  DRAFTING --> FAILED: PostFailed
  EDITING --> FAILED: PostFailed
  PUBLISHING --> FAILED: PostFailed
  PUBLISHED --> [*]
  FAILED --> [*]
```

`GuardrailBlocked` and `ContentCleared` are side-events recorded on the entity for audit; they do not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  PostEntity ||--o{ PostCreated : emits
  PostEntity ||--o{ ResearchStarted : emits
  PostEntity ||--o{ ResearchCollected : emits
  PostEntity ||--o{ OutlineStarted : emits
  PostEntity ||--o{ OutlineProduced : emits
  PostEntity ||--o{ DraftStarted : emits
  PostEntity ||--o{ DraftWritten : emits
  PostEntity ||--o{ EditStarted : emits
  PostEntity ||--o{ EditApplied : emits
  PostEntity ||--o{ PublishStarted : emits
  PostEntity ||--o{ ContentCleared : emits
  PostEntity ||--o{ PostPublished : emits
  PostEntity ||--o{ GuardrailBlocked : emits
  PostEntity ||--o{ PostFailed : emits
  PostView }o--|| PostEntity : projects
  BlogPipelineWorkflow }o--|| PostEntity : reads-and-writes
  BlogWriterAgent ||--o{ ResearchNotes : returns
  BlogWriterAgent ||--o{ Outline : returns
  BlogWriterAgent ||--o{ Draft : returns
  BlogWriterAgent ||--o{ EditedDraft : returns
  BlogWriterAgent ||--o{ Post : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PostEndpoint` | `api/PostEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PostEntity` | `application/PostEntity.java` (state in `domain/PostRecord.java`, events in `domain/PostEvent.java`) |
| `BlogPipelineWorkflow` | `application/BlogPipelineWorkflow.java` |
| `BlogWriterAgent` | `application/BlogWriterAgent.java` (tasks in `application/BlogTasks.java`) |
| `ResearchTools` | `application/ResearchTools.java` |
| `OutlineTools` | `application/OutlineTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `EditTools` | `application/EditTools.java` |
| `PublishTools` | `application/PublishTools.java` |
| `ContentPolicyGuardrail` | `application/ContentPolicyGuardrail.java` |
| `PostView` | `application/PostView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `researchStep` 90 s, `outlineStep` 90 s, `draftStep` 90 s, `editStep` 90 s, `publishStep` 90 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BlogPipelineWorkflow::error)`. The 90 s on each agent-calling step accommodates LLM latency including tool round-trips for Ollama's slower inference (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + postId` as the workflow id; restart of the same postId is rejected by the workflow runtime. The agent instance id is `"agent-" + postId` so each post has its own per-task conversation memory.
- **One agent per post**: `BlogWriterAgent` runs five tasks per post — RESEARCH, OUTLINE, DRAFT, EDIT, PUBLISH — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a non-compliant response and still let the agent self-correct.
- **Guardrail-driven retry**: when `ContentPolicyGuardrail` rejects a response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail policy, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Guardrail is synchronous and deterministic**: `ContentPolicyGuardrail` runs in-process. No LLM call — the same prose always produces the same policy decision. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `researchStep` writes `ResearchCollected` BEFORE advancing; `outlineStep` reads the recorded `ResearchNotes` from the entity to build its task's instruction context; `draftStep` reads both `ResearchNotes` and `Outline`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed post stays at the last successful event; the UI shows the partial state for the user.
