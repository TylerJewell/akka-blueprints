# PLAN — content-pipeline-subworkflow-orchestrator

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef subwf fill:#1a0e2a,stroke:#C084FC,color:#C084FC;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ContentEndpoint]:::ep
  Entity[ArticleEntity]:::ese
  WF[ContentPipelineWorkflow]:::wf
  Agent[ContentPipelineAgent]:::agent
  ResWF[ResearchSubworkflow]:::subwf
  DraftWF[DraftSubworkflow]:::subwf
  RevWF[ReviewSubworkflow]:::subwf
  ResTools[ResearchTools]:::tool
  DraftTools[DraftTools]:::tool
  RevTools[ReviewTools]:::tool
  Guard[ToneGuardrail]:::guard
  View[ArticleView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  API -->|resume decision| WF
  WF -->|researchStep runSingleTask| Agent
  WF -->|draftStep runSingleTask| Agent
  WF -->|reviewStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|invokes| ResTools
  Agent -->|invokes| DraftTools
  Agent -->|invokes| RevTools
  ResTools -->|starts| ResWF
  DraftTools -->|starts| DraftWF
  RevTools -->|starts| RevWF
  ResWF -->|ResearchFindings| ResTools
  DraftWF -->|Draft| DraftTools
  RevWF -->|ReviewResult| RevTools
  Agent -->|ResearchFindings / Draft / ReviewResult| WF
  WF -->|recordResearch/Draft/Review| Entity
  WF -->|guardStep check| Guard
  Guard -->|GuardResult| WF
  WF -->|awaitApprovalStep| Entity
  WF -->|publishStep/rejectStep| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path through PUBLISHED)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ContentEndpoint
  participant E as ArticleEntity
  participant W as ContentPipelineWorkflow
  participant A as ContentPipelineAgent
  participant G as ToneGuardrail
  participant SubWF as Sub-Workflows
  participant Ed as Editor (UI)

  U->>API: POST /api/articles { topic }
  API->>E: create(topic)
  E-->>API: { articleId }
  API->>W: start(articleId, topic)
  W->>E: startResearch
  W->>A: runSingleTask(RESEARCH_TOPIC, topic)
  A->>SubWF: ResearchTools → start ResearchSubworkflow
  SubWF-->>A: ResearchFindings
  A-->>W: ResearchFindings
  W->>E: recordResearch
  W->>A: runSingleTask(DRAFT_CONTENT, findings)
  A->>SubWF: DraftTools → start DraftSubworkflow
  SubWF-->>A: Draft
  A-->>W: Draft
  W->>E: recordDraft
  W->>A: runSingleTask(REVIEW_DRAFT, draft+findings)
  A->>SubWF: ReviewTools → start ReviewSubworkflow
  SubWF-->>A: ReviewResult
  A->>G: before-agent-response(ReviewResult)
  G-->>A: accept (no tone violations, all cited URLs in sources)
  A-->>W: ReviewResult
  W->>E: recordReview
  W->>G: guardStep.check(draft, findings)
  G-->>W: GuardResult{passed:true}
  W->>E: recordGuardResult + requestApproval
  E-.->>U: SSE event(AWAITING_APPROVAL)
  Ed->>API: POST /api/articles/{id}/decision { approved: true }
  API->>W: resume(EditorDecision)
  W->>E: editorApproved + articlePublished
  E-.->>U: SSE event(PUBLISHED)
```

## State machine — `ArticleEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RESEARCHING: ResearchStarted
  RESEARCHING --> RESEARCHED: ResearchCompleted
  RESEARCHED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: DraftCompleted
  DRAFTED --> REVIEWING: ReviewStarted
  REVIEWING --> REVIEWED: ReviewCompleted
  REVIEWED --> AWAITING_APPROVAL: ApprovalRequested
  AWAITING_APPROVAL --> APPROVED: EditorApproved
  APPROVED --> PUBLISHED: ArticlePublished
  AWAITING_APPROVAL --> REJECTED: EditorRejected
  RESEARCHING --> FAILED: ArticleFailed
  DRAFTING --> FAILED: ArticleFailed
  REVIEWING --> FAILED: ArticleFailed
  PUBLISHED --> [*]
  REJECTED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  ArticleEntity ||--o{ ArticleCreated : emits
  ArticleEntity ||--o{ ResearchStarted : emits
  ArticleEntity ||--o{ ResearchCompleted : emits
  ArticleEntity ||--o{ DraftStarted : emits
  ArticleEntity ||--o{ DraftCompleted : emits
  ArticleEntity ||--o{ ReviewStarted : emits
  ArticleEntity ||--o{ ReviewCompleted : emits
  ArticleEntity ||--o{ GuardChecked : emits
  ArticleEntity ||--o{ ApprovalRequested : emits
  ArticleEntity ||--o{ EditorApproved : emits
  ArticleEntity ||--o{ EditorRejected : emits
  ArticleEntity ||--o{ ArticlePublished : emits
  ArticleEntity ||--o{ GuardrailRejected : emits
  ArticleEntity ||--o{ ArticleFailed : emits
  ArticleView }o--|| ArticleEntity : projects
  ContentPipelineWorkflow }o--|| ArticleEntity : reads-and-writes
  ResearchSubworkflow ||--o{ ResearchFindings : returns
  DraftSubworkflow ||--o{ Draft : returns
  ReviewSubworkflow ||--o{ ReviewResult : returns
  ContentPipelineAgent ||--o{ ResearchFindings : task-result
  ContentPipelineAgent ||--o{ Draft : task-result
  ContentPipelineAgent ||--o{ ReviewResult : task-result
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ContentEndpoint` | `api/ContentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ArticleEntity` | `application/ArticleEntity.java` (state in `domain/ArticleRecord.java`, events in `domain/ArticleEvent.java`) |
| `ContentPipelineWorkflow` | `application/ContentPipelineWorkflow.java` |
| `ResearchSubworkflow` | `application/ResearchSubworkflow.java` |
| `DraftSubworkflow` | `application/DraftSubworkflow.java` |
| `ReviewSubworkflow` | `application/ReviewSubworkflow.java` |
| `ContentPipelineAgent` | `application/ContentPipelineAgent.java` (tasks in `application/ContentPipelineTasks.java`) |
| `ResearchTools` | `application/ResearchTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `ReviewTools` | `application/ReviewTools.java` |
| `ToneGuardrail` | `application/ToneGuardrail.java` |
| `ArticleView` | `application/ArticleView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `researchStep` 90 s, `draftStep` 90 s, `reviewStep` 90 s, `guardStep` 10 s, `awaitApprovalStep` no timeout (waits for editor), `publishStep` 5 s, `rejectStep` 5 s, `error` 5 s. Sub-workflow steps: ResearchSubworkflow 15 s each, DraftSubworkflow 15 s each, ReviewSubworkflow 10 s each.
- **Idempotency**: each main workflow uses `"pipeline-" + articleId` as its id; each sub-workflow uses `"research-" + articleId`, `"draft-" + articleId`, `"review-" + articleId`. Restart of the same articleId is rejected by the workflow runtime.
- **One agent per article**: `ContentPipelineAgent` instance id is `"agent-" + articleId`. Each task runs with `maxIterationsPerTask(4)`.
- **Guardrail-driven retry**: `ToneGuardrail` rejects non-compliant agent responses; the loop retries within 4 iterations. If all 4 fail, the step fails over to `error` and the article transitions to `FAILED`.
- **Sub-workflow transparency**: sub-workflows are implementation artifacts. From the sequential-pipeline's perspective, each tool call returns a typed value; whether that value came from a simple function or a multi-step workflow is invisible to the agent and to the pipeline's dependency contract.
- **HITL hard gate**: `awaitApprovalStep` is the only path to `publishStep`. No agent action or timeout can bypass it. The article sits at `AWAITING_APPROVAL` indefinitely until an editor decision arrives.
- **No saga / no compensation**: every step is append-only. A `FAILED` or `REJECTED` article retains its partial phase data; the UI shows what completed before the terminal event.
