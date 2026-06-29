# PLAN — composed-hiring-workflow

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab with the Akka theme variables and the Lesson 24 state-label CSS overrides.

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

  HM[HiringManager]:::agent
  SLead[ScreeningLead]:::agent
  Scr[Screener]:::agent
  Coach[CvCoach]:::agent
  Critic[CvCritic]:::agent
  Inter[Interviewer]:::agent

  HTW[HiringTeamWorkflow]:::wf
  CandWF[CandidateWorkflow]:::wf
  CvLoop[CvImprovementLoop]:::wf

  App[ApplicationEntity]:::ese
  Cv[CvEntity]:::ese
  Queue[ApplicationQueue]:::ese
  ABoard[ApplicationBoardView]:::view
  CvBoard[CvBoardView]:::view
  AppC[ApplicationRequestConsumer]:::cons
  EvalC[StageEvalConsumer]:::cons
  Sim[ApplicationSimulator]:::ta
  Stale[StaleDimensionMonitor]:::ta
  API[HiringEndpoint]:::ep
  AppEP[AppEndpoint]:::ep

  API -->|submit application| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| AppC
  AppC -->|create| App
  AppC -->|start workflow| HTW

  HTW -->|OPEN_REQ / DRAFT_OFFER| HM
  HTW -->|starts nested| CandWF
  HTW -->|starts nested| CvLoop
  HTW -->|INTERVIEW_AXIS x axes| Inter
  HTW -->|record stages| App

  CandWF -->|PLAN_SCREENING / SYNTHESIZE| SLead
  CandWF -->|SCREEN_DIMENSION x N| Scr
  CandWF -->|record report| App

  CvLoop -->|IMPROVE_CV x iterations| Coach
  CvLoop -->|CRITIQUE_CV x iterations| Critic
  CvLoop -->|add / accept draft| Cv

  Scr -.->|ApplicationTools write| App
  Coach -.->|ApplicationTools write| Cv
  Inter -.->|ApplicationTools write| App

  App -.->|projects| ABoard
  Cv -.->|projects| CvBoard
  App -.->|stage events| EvalC
  EvalC -->|record eval| App
  Stale -.->|every 60s| App

  API -->|query / SSE| ABoard
  API -->|query / SSE| CvBoard
  API -->|post-hire review| App
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions, scheduled ticks, and guarded tool writes. `Screener` and `Interviewer` are each one agent class run as several instances. The `HiringTeamWorkflow` is the top-level orchestrator; it nests two workflows — `CandidateWorkflow` (screening delegation) and `CvImprovementLoop` (coach-critic feedback) — before running the interview panel (moderation).

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as HiringEndpoint
  participant Q as ApplicationQueue
  participant C as ApplicationRequestConsumer
  participant HTW as HiringTeamWorkflow
  participant HM as HiringManager
  participant CWF as CandidateWorkflow
  participant SL as ScreeningLead
  participant SC as Screener
  participant CIL as CvImprovementLoop
  participant CC as CvCoach
  participant CK as CvCritic
  participant IV as Interviewer
  participant AppE as ApplicationEntity
  participant CvE as CvEntity

  U->>API: POST /api/applications {candidateName, jobRole, cvText}
  API->>Q: submitApplication(brief)
  API-->>U: 202 {applicationId}
  Q->>C: ApplicationSubmitted
  C->>AppE: createApplication
  C->>HTW: start({applicationId})
  HTW->>HM: OPEN_REQ(jobRole, cvText)
  HM-->>HTW: HiringBrief
  HTW->>AppE: openBrief
  HTW->>CWF: start nested CandidateWorkflow
  CWF->>SL: PLAN_SCREENING
  SL-->>CWF: ScreeningPlan{dimensions}
  CWF->>SC: SCREEN_DIMENSION x N
  SC-->>AppE: appendScreeningNote (guarded)
  CWF->>SL: SYNTHESIZE_SCREENING(notes)
  SL-->>CWF: ScreeningReport
  CWF->>AppE: recordScreeningReport
  CWF-->>HTW: ScreeningReport (nested complete)
  HTW->>CIL: start nested CvImprovementLoop
  CIL->>CC: IMPROVE_CV(originalCv, iteration=1)
  CC-->>CvE: addDraft (guarded)
  CIL->>CK: CRITIQUE_CV(draft)
  CK-->>CIL: CritiqueNote{outcome=PASS}
  CIL->>CvE: acceptDraft
  CIL-->>HTW: CvDraft (nested complete)
  HTW->>AppE: recordImprovedCv
  HTW->>IV: INTERVIEW_AXIS(technical, draft)
  HTW->>IV: INTERVIEW_AXIS(behavioural, draft)
  HTW->>IV: INTERVIEW_AXIS(cultural, draft)
  IV-->>AppE: appendInterviewScore (guarded)
  HTW->>HTW: PanelRule(scores) -> PROCEED
  HTW->>AppE: recordPanelVerdict(PROCEED)
  HTW->>HM: DRAFT_OFFER(candidate, role)
  Note over HM: before-agent-response guardrail vets OfferLetter
  HM-->>HTW: OfferLetter
  HTW->>AppE: extendOffer(letter, ref)
  AppE-->>API: SSE status=OFFER_EXTENDED
```

## State machine — `ApplicationEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED: createApplication
  SUBMITTED --> OPENED: openBrief
  OPENED --> SCREENING: CandidateWorkflow starts
  SCREENING --> SCREENED: recordScreeningReport
  SCREENED --> CV_IMPROVING: CvImprovementLoop starts
  CV_IMPROVING --> CV_IMPROVED: recordImprovedCv
  CV_IMPROVED --> INTERVIEWING: panel runs
  INTERVIEWING --> SHORTLISTED: recordPanelVerdict PROCEED
  INTERVIEWING --> CV_IMPROVING: requestReassess (one bounded round)
  SHORTLISTED --> OFFER_PENDING: DRAFT_OFFER called
  OFFER_PENDING --> OFFER_EXTENDED: extendOffer (output guardrail passes)
  OFFER_PENDING --> OFFER_PENDING: recordOfferBlock (guardrail refusal)
  OFFER_EXTENDED --> OFFER_EXTENDED: recordPostHireReview (post-hire, non-blocking)
  INTERVIEWING --> DECLINED: recordPanelVerdict DECLINE
  OFFER_EXTENDED --> [*]
  DECLINED --> [*]
```

## Entity model

```mermaid
erDiagram
  ApplicationEntity ||--o{ ApplicationCreated : emits
  ApplicationEntity ||--o{ BriefOpened : emits
  ApplicationEntity ||--o{ ScreeningCompleted : emits
  ApplicationEntity ||--o{ CvImproved : emits
  ApplicationEntity ||--o{ InterviewCompleted : emits
  ApplicationEntity ||--o{ ReassessRequested : emits
  ApplicationEntity ||--o{ OfferDrafted : emits
  ApplicationEntity ||--o{ OfferExtended : emits
  ApplicationEntity ||--o{ OfferBlocked : emits
  ApplicationEntity ||--o{ StageEvaluated : emits
  ApplicationEntity ||--o{ PostHireReviewRecorded : emits
  ApplicationBoardView }o--|| ApplicationEntity : projects
  StageEvalConsumer }o--|| ApplicationEntity : subscribes
  CvEntity ||--o{ CvCreated : emits
  CvEntity ||--o{ DraftAdded : emits
  CvEntity ||--o{ DraftAccepted : emits
  CvBoardView }o--|| CvEntity : projects
  ApplicationEntity ||--|| CvEntity : "one CV per application"
  ApplicationQueue ||--o{ ApplicationSubmitted : emits
  ApplicationRequestConsumer }o--|| ApplicationQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `HiringManager` | `application/HiringManager.java` |
| `ScreeningLead` | `application/ScreeningLead.java` |
| `Screener` | `application/Screener.java` |
| `CvCoach` | `application/CvCoach.java` |
| `CvCritic` | `application/CvCritic.java` |
| `Interviewer` | `application/Interviewer.java` |
| `HiringTasks` | `application/HiringTasks.java` |
| `ApplicationTools` | `application/ApplicationTools.java` |
| `PanelRule` | `application/PanelRule.java` |
| `StageEvaluator` | `application/StageEvaluator.java` |
| `HiringTeamWorkflow` | `application/HiringTeamWorkflow.java` |
| `CandidateWorkflow` | `application/CandidateWorkflow.java` |
| `CvImprovementLoop` | `application/CvImprovementLoop.java` |
| `ApplicationEntity` | `application/ApplicationEntity.java` (state in `domain/Application.java`, events in `domain/ApplicationEvent.java`) |
| `CvEntity` | `application/CvEntity.java` (state in `domain/CvIteration.java`, events in `domain/CvEvent.java`) |
| `ApplicationQueue` | `application/ApplicationQueue.java` |
| `ApplicationBoardView` | `application/ApplicationBoardView.java` |
| `CvBoardView` | `application/CvBoardView.java` |
| `ApplicationRequestConsumer` | `application/ApplicationRequestConsumer.java` |
| `StageEvalConsumer` | `application/StageEvalConsumer.java` |
| `ApplicationSimulator` | `application/ApplicationSimulator.java` |
| `StaleDimensionMonitor` | `application/StaleDimensionMonitor.java` |
| `HiringEndpoint` | `api/HiringEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **6 autonomous-agent · 3 workflow · 3 event-sourced-entity · 2 view · 2 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Three coordination capabilities under one pipeline.** `HiringTeamWorkflow` is a sequential orchestrator; it delegates to two nested workflows before running the moderation panel. Each nested workflow has its own state and lifecycle separate from the outer workflow.
- **Nested workflows are bounded sub-pipelines.** `CandidateWorkflow` runs the screening delegation and returns a `ScreeningReport`; `CvImprovementLoop` runs the coach-critic cycle and returns the accepted `CvDraft`. The top-level workflow awaits each nested workflow's completion before proceeding.
- **The coach-critic loop is bounded.** `CvImprovementLoop` increments an `iterationCount` on each pass; at 3 iterations or on a `PASS` critique it accepts the current draft and ends. The pipeline always terminates.
- **Moderation panel under contention.** Three `Interviewer` instances run against the same application concurrently. Each writes its `InterviewScore` via `ApplicationTools`, which is gated by the G1 before-agent-invocation guardrail. `PanelRule` is a deterministic pure function — no LLM call — so the verdict is reproducible.
- **Workflow step timeouts.** Every step that calls an agent sets an explicit `stepTimeout` (Lesson 4) — `openStep` 60 s, `screenStep` 180 s (nests `CandidateWorkflow`), `improveStep` 180 s (nests `CvImprovementLoop`), `interviewStep` 120 s (fans out three interviewer calls), `offerStep` 60 s. `coachStep` and `criticStep` inside `CvImprovementLoop` each set 90 s and 60 s respectively.
- **The output guardrail can stall, not crash.** If the G2 before-agent-response guardrail refuses the `OfferLetter`, `offerStep` records the block and ends with the application left `OFFER_PENDING`; nothing is extended and the reason is visible in the UI.
- **Release for liveness.** `StaleDimensionMonitor` resets a screening dimension idle for more than three minutes to `OPEN` so a failed screener does not strand the pipeline.
- **The stage eval is downstream and non-blocking.** `StageEvalConsumer` subscribes to `ApplicationEntity` events and records a `StageEval` after each stage result lands; it never gates the pipeline (control E1).
- **Post-hire review is on the loop.** `recordPostHireReview` is accepted only when the application is `OFFER_EXTENDED` and never changes that status (control HO1).
- **Idempotency.** Deterministic `dimensionId = applicationId + "-d" + index` makes `appendScreeningDimension` idempotent if `planStep` is retried; `applicationId` is the `HiringTeamWorkflow` id so a redelivered `ApplicationSubmitted` starts the same workflow, not a duplicate.
