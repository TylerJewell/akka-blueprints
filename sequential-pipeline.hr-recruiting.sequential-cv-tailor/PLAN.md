# PLAN — sequential-cv-tailor

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
  classDef sanitizer fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[CvEndpoint]:::ep
  Entity[CvRequestEntity]:::ese
  WF[CvPipelineWorkflow]:::wf
  GenAgent[CvGeneratorAgent]:::agent
  TailAgent[CvTailorAgent]:::agent
  ProfTools[ProfileTools]:::tool
  TailTools[TailorTools]:::tool
  Sanitizer[PiiSanitizer]:::sanitizer
  Scorer[AlignmentScorer]:::sanitizer
  View[CvRequestView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|generateStep runSingleTask| GenAgent
  GenAgent -.->|before-call| Sanitizer
  GenAgent -->|invokes| ProfTools
  Sanitizer -->|SanitizationApplied| Entity
  GenAgent -->|BaseCv| WF
  WF -->|recordBaseCv| Entity
  WF -->|tailorStep runSingleTask| TailAgent
  TailAgent -.->|before-call| Sanitizer
  TailAgent -->|invokes| TailTools
  TailAgent -->|TailoredCv| WF
  WF -->|recordTailoredCv| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|AlignmentResult| WF
  WF -->|recordAlignment| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant R as Recruiter (UI)
  participant API as CvEndpoint
  participant E as CvRequestEntity
  participant W as CvPipelineWorkflow
  participant S as PiiSanitizer
  participant GA as CvGeneratorAgent
  participant PT as ProfileTools
  participant TA as CvTailorAgent
  participant TT as TailorTools
  participant Sc as AlignmentScorer

  R->>API: POST /api/cv-requests { candidateProfileId, jobPostingId }
  API->>E: create(candidateProfileId, jobPostingId)
  E-->>API: { requestId }
  API->>W: start(requestId)
  W->>E: startGeneration
  W->>GA: runSingleTask(GENERATE_BASE_CV, profile)
  GA->>S: before-call(payload with PII)
  S-->>GA: scrubbed payload
  S->>E: recordSanitization(removedCount)
  GA->>PT: expandSummary + extractSkills + formatExperience
  PT-->>GA: summary / skills / experience
  GA-->>W: BaseCv
  W->>E: recordBaseCv
  W->>TA: runSingleTask(TAILOR_CV, baseCv + posting)
  TA->>S: before-call(payload — BaseCv only, no raw profile)
  S-->>TA: scrubbed payload (0 PII fields found)
  TA->>TT: matchKeywords + rewriteSummary + adjustExperience
  TT-->>TA: matches / tailoredSummary / adjustedExperience
  TA-->>W: TailoredCv
  W->>E: recordTailoredCv
  W->>Sc: score(tailoredCv, jobPosting)
  Sc-->>W: AlignmentResult
  W->>E: recordAlignment
  E-.->>R: SSE event(SCORED)
```

## State machine — `CvRequestEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> GENERATING: GenerationStarted
  GENERATING --> GENERATED: BaseCvReady
  GENERATED --> TAILORING: TailoringStarted
  TAILORING --> TAILORED: TailoredCvReady
  TAILORED --> SCORED: QualityScored
  GENERATING --> FAILED: RequestFailed
  TAILORING --> FAILED: RequestFailed
  SCORED --> [*]
  FAILED --> [*]
```

`SanitizationApplied` is a side-event recorded for audit; it does not change the status. The sanitizer may fire on the same request multiple times (once per model call). Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  CvRequestEntity ||--o{ CvRequestCreated : emits
  CvRequestEntity ||--o{ GenerationStarted : emits
  CvRequestEntity ||--o{ BaseCvReady : emits
  CvRequestEntity ||--o{ TailoringStarted : emits
  CvRequestEntity ||--o{ TailoredCvReady : emits
  CvRequestEntity ||--o{ QualityScored : emits
  CvRequestEntity ||--o{ SanitizationApplied : emits
  CvRequestEntity ||--o{ RequestFailed : emits
  CvRequestView }o--|| CvRequestEntity : projects
  CvPipelineWorkflow }o--|| CvRequestEntity : reads-and-writes
  CvGeneratorAgent ||--o{ BaseCv : returns
  CvTailorAgent ||--o{ TailoredCv : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CvEndpoint` | `api/CvEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CvRequestEntity` | `application/CvRequestEntity.java` (state in `domain/CvRequestRecord.java`, events in `domain/CvRequestEvent.java`) |
| `CvPipelineWorkflow` | `application/CvPipelineWorkflow.java` |
| `CvGeneratorAgent` | `application/CvGeneratorAgent.java` (tasks in `application/CvTasks.java`) |
| `CvTailorAgent` | `application/CvTailorAgent.java` |
| `ProfileTools` | `application/ProfileTools.java` |
| `TailorTools` | `application/TailorTools.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `AlignmentScorer` | `application/AlignmentScorer.java` |
| `CvRequestView` | `application/CvRequestView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `generateStep` 60 s, `tailorStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CvPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"cv-pipeline-" + requestId` as the workflow id; restart of the same requestId is rejected by the workflow runtime. Generator agent id `"gen-" + requestId`, tailor agent id `"tailor-" + requestId`, so each request has its own per-task conversation memory.
- **Two agents, one request**: `CvGeneratorAgent` runs one task per request (`GENERATE_BASE_CV`); `CvTailorAgent` runs one task per request (`TAILOR_CV`). Each has `maxIterationsPerTask(4)`.
- **Sanitizer is stateless**: `PiiSanitizer` holds no mutable state; registering it on two agents is safe. The requestId comes from `TaskDef.metadata("requestId", ...)`.
- **Eval is synchronous and deterministic**: `AlignmentScorer` runs in-process inside `evalStep`. No LLM call — same `TailoredCv` always scores the same.
- **Typed handoff is the data boundary**: `generateStep` writes `BaseCvReady` before returning; `tailorStep` reads the recorded `BaseCv` from the entity to build its task instruction context. `CvTailorAgent` never receives the raw `CandidateProfile`.
- **No saga / no compensation**: a failed request stays at its last successful event; the UI shows the partial state.
