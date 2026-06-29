# PLAN — blog-writer-pipeline

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

  API[BlogPostEndpoint]:::ep
  Entity[BlogPostEntity]:::ese
  WF[BlogWritingWorkflow]:::wf
  Agent[BlogWriterAgent]:::agent
  Research[ResearchTools]:::tool
  Outline[OutlineTools]:::tool
  Draft[DraftTools]:::tool
  Guard[BrandGuardrail]:::guard
  Scorer[QualityScorer]:::guard
  View[BlogPostView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|researchStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|invokes| Research
  Agent -->|invokes| Outline
  Agent -->|invokes| Draft
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|ResearchNotes / Outline / BlogPost| WF
  WF -->|recordResearch/Outline/Draft| Entity
  WF -->|qualityCheckStep score| Scorer
  Scorer -->|QualityResult| WF
  WF -->|recordQuality| Entity
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
  participant API as BlogPostEndpoint
  participant E as BlogPostEntity
  participant W as BlogWritingWorkflow
  participant A as BlogWriterAgent
  participant G as BrandGuardrail
  participant T as Tools (Research/Outline/Draft)
  participant Sc as QualityScorer

  U->>API: POST /api/posts { topic, style }
  API->>E: create(topic, style)
  E-->>API: { postId }
  API->>W: start(postId, topic, style)
  W->>E: startResearch
  W->>A: runSingleTask(RESEARCH_TOPIC, topic)
  A->>T: searchTopicReferences + fetchKeyPoints
  T-->>A: List<Reference>
  A-->>W: ResearchNotes
  W->>E: recordResearch
  W->>A: runSingleTask(OUTLINE_POST, notes)
  A->>T: generateSectionHeadings + assignKeyPoints
  T-->>A: List<OutlineSection>
  A-->>W: Outline
  W->>E: recordOutline
  W->>A: runSingleTask(DRAFT_POST, outline + notes)
  A->>T: writeSection + writeConclusion
  T-->>A: PostSection / String
  A->>G: before-agent-response(BlogPost candidate)
  G-->>A: accept (word count >= 300, no forbidden phrases, CTA present)
  A-->>W: BlogPost
  W->>E: recordDraft
  W->>Sc: score(post, outline, notes)
  Sc-->>W: QualityResult
  W->>E: recordQuality
  E-.->>U: SSE event(QUALITY_CHECKED)
```

## State machine — `BlogPostEntity`

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
  DRAFTED --> QUALITY_CHECKED: QualityChecked
  RESEARCHING --> FAILED: PostFailed
  OUTLINING --> FAILED: PostFailed
  DRAFTING --> FAILED: PostFailed
  QUALITY_CHECKED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same `draftStep`, and the workflow step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  BlogPostEntity ||--o{ PostCreated : emits
  BlogPostEntity ||--o{ ResearchStarted : emits
  BlogPostEntity ||--o{ ResearchCollected : emits
  BlogPostEntity ||--o{ OutlineStarted : emits
  BlogPostEntity ||--o{ OutlineProduced : emits
  BlogPostEntity ||--o{ DraftStarted : emits
  BlogPostEntity ||--o{ DraftWritten : emits
  BlogPostEntity ||--o{ QualityChecked : emits
  BlogPostEntity ||--o{ GuardrailRejected : emits
  BlogPostEntity ||--o{ PostFailed : emits
  BlogPostView }o--|| BlogPostEntity : projects
  BlogWritingWorkflow }o--|| BlogPostEntity : reads-and-writes
  BlogWriterAgent ||--o{ ResearchNotes : returns
  BlogWriterAgent ||--o{ Outline : returns
  BlogWriterAgent ||--o{ BlogPost : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BlogPostEndpoint` | `api/BlogPostEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BlogPostEntity` | `application/BlogPostEntity.java` (state in `domain/BlogPostRecord.java`, events in `domain/BlogPostEvent.java`) |
| `BlogWritingWorkflow` | `application/BlogWritingWorkflow.java` |
| `BlogWriterAgent` | `application/BlogWriterAgent.java` (tasks in `application/BlogTasks.java`) |
| `ResearchTools` | `application/ResearchTools.java` |
| `OutlineTools` | `application/OutlineTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `BrandGuardrail` | `application/BrandGuardrail.java` |
| `QualityScorer` | `application/QualityScorer.java` |
| `BlogPostView` | `application/BlogPostView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `researchStep` 60 s, `outlineStep` 60 s, `draftStep` 90 s (accommodates guardrail retry iterations), `qualityCheckStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BlogWritingWorkflow::error)`. The 90 s on `draftStep` accommodates LLM latency plus up to two brand-guardrail rejection-and-retry cycles within the 4-iteration budget (Lesson 4).
- **Idempotency**: each workflow uses `"workflow-" + postId` as the workflow id; restart of the same postId is rejected by the workflow runtime. The agent instance id is `"agent-" + postId` so each post has its own per-task conversation memory.
- **One agent per post**: `BlogWriterAgent` runs three tasks per post — RESEARCH, OUTLINE, DRAFT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the brand guardrail room to reject a non-conformant draft and still let the agent self-correct.
- **Guardrail-driven retry**: when `BrandGuardrail` rejects a response, the rejection is returned as a structured error to the agent loop naming the failing rule. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail brand checks, the workflow step fails over to `error` and the entity transitions to FAILED.
- **Eval is synchronous and deterministic**: `QualityScorer` runs in-process inside `qualityCheckStep`. No LLM call, no external service — the same post always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `researchStep` writes `ResearchCollected` BEFORE returning; `outlineStep` reads the recorded `ResearchNotes` from the entity to build its task's instruction context; `draftStep` reads both `ResearchNotes` and `Outline`. The agent itself is stateless across phases — it never holds research + outline + draft context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed post stays at the last successful event; the UI shows the partial state for the user.
