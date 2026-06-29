# PLAN — video-to-blog-pipeline

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

  API[BlogEndpoint]:::ep
  Entity[BlogPostEntity]:::ese
  WF[BlogPipelineWorkflow]:::wf
  Agent[BlogAgent]:::agent
  TxTools[TranscriptTools]:::tool
  SumTools[SummaryTools]:::tool
  DraftTools[DraftTools]:::tool
  PolishTools[PolishTools]:::tool
  Guard[PublishGuardrail]:::guard
  Scorer[EditorialScorer]:::guard
  View[BlogPostView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|transcriptStep runSingleTask| Agent
  WF -->|summaryStep runSingleTask| Agent
  WF -->|draftStep runSingleTask| Agent
  WF -->|polishStep runSingleTask| Agent
  Agent -->|invokes| TxTools
  Agent -->|invokes| SumTools
  Agent -->|invokes| DraftTools
  Agent -->|invokes| PolishTools
  Agent -.->|before-agent-response| Guard
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|Transcript / VideoSummary / BlogDraft / BlogPost| WF
  WF -->|recordTranscript/Summary/Draft/Post| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
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
  participant API as BlogEndpoint
  participant E as BlogPostEntity
  participant W as BlogPipelineWorkflow
  participant A as BlogAgent
  participant T as Tools (Transcript/Summary/Draft/Polish)
  participant G as PublishGuardrail
  participant Sc as EditorialScorer

  U->>API: POST /api/posts { videoUrl }
  API->>E: create(videoUrl)
  E-->>API: { postId }
  API->>W: start(postId, videoUrl)
  W->>E: startTranscript
  W->>A: runSingleTask(EXTRACT_TRANSCRIPT, videoUrl)
  A->>T: fetchTranscript + extractChapters
  T-->>A: Transcript
  A-->>W: Transcript
  W->>E: recordTranscript
  W->>A: runSingleTask(SUMMARISE_VIDEO, transcript)
  A->>T: extractKeyPoints + outlineSections
  T-->>A: VideoSummary
  A-->>W: VideoSummary
  W->>E: recordSummary
  W->>A: runSingleTask(DRAFT_POST, summary)
  A->>T: composeIntroduction + writeSectionBody
  T-->>A: BlogDraft
  A-->>W: BlogDraft
  W->>E: recordDraft
  W->>A: runSingleTask(POLISH_POST, draft)
  A->>T: polishProse + writeConclusion
  T-->>A: BlogPost (response)
  A->>G: before-agent-response(BlogPost text)
  G-->>A: accept (no prohibited content)
  A-->>W: BlogPost
  W->>E: recordPost
  W->>Sc: score(post, draft, summary)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `BlogPostEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> TRANSCRIBING: TranscriptStarted
  TRANSCRIBING --> TRANSCRIBED: TranscriptExtracted
  TRANSCRIBED --> SUMMARISING: SummaryStarted
  SUMMARISING --> SUMMARISED: SummaryProduced
  SUMMARISED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: DraftWritten
  DRAFTED --> POLISHING: PolishStarted
  POLISHING --> POLISHED: PostPolished
  POLISHED --> EVALUATED: EvaluationScored
  TRANSCRIBING --> FAILED: PostFailed
  SUMMARISING --> FAILED: PostFailed
  DRAFTING --> FAILED: PostFailed
  POLISHING --> FAILED: PostFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same POLISH task, and the workflow's polishStep continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  BlogPostEntity ||--o{ PostCreated : emits
  BlogPostEntity ||--o{ TranscriptStarted : emits
  BlogPostEntity ||--o{ TranscriptExtracted : emits
  BlogPostEntity ||--o{ SummaryStarted : emits
  BlogPostEntity ||--o{ SummaryProduced : emits
  BlogPostEntity ||--o{ DraftStarted : emits
  BlogPostEntity ||--o{ DraftWritten : emits
  BlogPostEntity ||--o{ PolishStarted : emits
  BlogPostEntity ||--o{ PostPolished : emits
  BlogPostEntity ||--o{ EvaluationScored : emits
  BlogPostEntity ||--o{ GuardrailRejected : emits
  BlogPostEntity ||--o{ PostFailed : emits
  BlogPostView }o--|| BlogPostEntity : projects
  BlogPipelineWorkflow }o--|| BlogPostEntity : reads-and-writes
  BlogAgent ||--o{ Transcript : returns
  BlogAgent ||--o{ VideoSummary : returns
  BlogAgent ||--o{ BlogDraft : returns
  BlogAgent ||--o{ BlogPost : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BlogEndpoint` | `api/BlogEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BlogPostEntity` | `application/BlogPostEntity.java` (state in `domain/BlogPostRecord.java`, events in `domain/BlogPostEvent.java`) |
| `BlogPipelineWorkflow` | `application/BlogPipelineWorkflow.java` |
| `BlogAgent` | `application/BlogAgent.java` (tasks in `application/BlogTasks.java`) |
| `TranscriptTools` | `application/TranscriptTools.java` |
| `SummaryTools` | `application/SummaryTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `PolishTools` | `application/PolishTools.java` |
| `PublishGuardrail` | `application/PublishGuardrail.java` |
| `EditorialScorer` | `application/EditorialScorer.java` |
| `BlogPostView` | `application/BlogPostView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `transcriptStep` 90 s, `summaryStep` 60 s, `draftStep` 60 s, `polishStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BlogPipelineWorkflow::error)`. The 90 s on `transcriptStep` accommodates the transcript-fetch round-trip; 60 s on each subsequent agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + postId` as the workflow id; restart of the same postId is rejected by the workflow runtime. The agent instance id is `"agent-" + postId` so each post has its own per-task conversation memory.
- **One agent per post**: `BlogAgent` runs four tasks per post — TRANSCRIPT, SUMMARISE, DRAFT, POLISH — each with `capability(...).maxIterationsPerTask(3)`. The 3-iteration budget on the POLISH task gives the guardrail room to reject a prohibited-content response and still let the agent self-correct.
- **Guardrail-driven retry**: when `PublishGuardrail` rejects a POLISH response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations are rejected, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `EditorialScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same post always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `transcriptStep` writes `TranscriptExtracted` BEFORE returning; `summaryStep` reads the recorded `Transcript` from the entity to build its task's instruction context; `draftStep` reads `VideoSummary`; `polishStep` reads `BlogDraft`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed post stays at the last successful event; the UI shows the partial state for the user.
