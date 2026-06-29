# PLAN — earnings-reviewer

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

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

  Planner[ReviewPlannerAgent]:::agent
  Trans[TranscriptReaderAgent]:::agent
  File[FilingReaderAgent]:::agent
  Model[ModelUpdaterAgent]:::agent
  Flag[ThesisFlagAgent]:::agent

  WF[ReviewWorkflow]:::wf
  Review[ReviewEntity]:::ese
  ModelE[ModelEntity]:::ese
  Queue[SubmissionQueue]:::ese
  View[ReviewView]:::view
  Consumer[ReviewSubmissionConsumer]:::cons
  Sim[EarningsSimulator]:::ta
  Stale[StaleReviewMonitor]:::ta
  API[ReviewEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / DECIDE / COMPOSE| Planner
  WF -->|READ_TRANSCRIPT| Trans
  WF -->|READ_FILING| File
  WF -->|UPDATE_MODEL| Model
  WF -->|RAISE_FLAGS| Flag
  WF -->|emit events| Review
  WF -->|apply diff| ModelE
  Review -.->|projects| View
  API -->|query/SSE| View
  API -->|get| ModelE
  Stale -.->|every 60s| Review
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ReviewEndpoint
  participant Q as SubmissionQueue
  participant C as ReviewSubmissionConsumer
  participant W as ReviewWorkflow
  participant P as ReviewPlannerAgent
  participant S as Specialist (Transcript/Filing/Model/Flag)
  participant E as ReviewEntity
  participant M as ModelEntity
  participant V as ReviewView

  U->>API: POST /api/reviews {ticker, period}
  API->>Q: append ReviewSubmitted
  API-->>U: 202 {reviewId}
  Q->>C: ReviewSubmitted
  C->>W: start({reviewId, ticker, period})
  W->>E: emit ReviewCreated (PENDING)
  W->>P: PLAN(ticker, period)
  P-->>W: ReviewLedger
  W->>E: emit ReviewPlanned, status READING
  loop until Complete | Fail | Stale
    W->>P: DECIDE(ledger)
    P-->>W: Continue(DispatchDecision)
    W->>S: runSingleTask(focus)
    S-->>W: SpecialistResult
    W->>E: emit PassagesAppended / ModelUpdatesAppended / FlagsAppended
  end
  W->>P: COMPOSE_ANSWER
  P-->>W: ReviewAnswer (candidate flags)
  loop for each candidate flag
    W->>W: FlagGuardrail.vet(flag, ledger)
    W->>E: emit FlagAccepted or FlagRejected
    W->>W: EvidenceQualityEvaluator.score(flag, ledger)
    W->>E: emit FlagScored
  end
  W->>M: updateMetric per accepted ModelUpdate
  W->>E: emit ReviewCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ReviewEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> READING: ReviewPlanned
  READING --> MODELING: ModelUpdatesAppended
  MODELING --> FLAGGING: FlagsAppended
  FLAGGING --> FINALIZING: COMPOSE_ANSWER returned
  FINALIZING --> COMPLETED: ReviewCompleted
  READING --> FAILED: ReviewFailed
  MODELING --> FAILED: ReviewFailed
  FLAGGING --> FAILED: ReviewFailed
  READING --> STALE: ReviewFailedStale
  MODELING --> STALE: ReviewFailedStale
  FLAGGING --> STALE: ReviewFailedStale
  COMPLETED --> [*]
  FAILED --> [*]
  STALE --> [*]
```

## Entity model

```mermaid
erDiagram
  ReviewEntity ||--o{ ReviewPlanned : emits
  ReviewEntity ||--o{ DispatchProposed : emits
  ReviewEntity ||--o{ PassagesAppended : emits
  ReviewEntity ||--o{ ModelUpdatesAppended : emits
  ReviewEntity ||--o{ FlagsAppended : emits
  ReviewEntity ||--o{ LedgerRevised : emits
  ReviewEntity ||--o{ FlagAccepted : emits
  ReviewEntity ||--o{ FlagRejected : emits
  ReviewEntity ||--o{ FlagScored : emits
  ReviewEntity ||--o{ ReviewCompleted : emits
  ReviewEntity ||--o{ ReviewFailed : emits
  ReviewEntity ||--o{ ReviewFailedStale : emits
  ReviewView }o--|| ReviewEntity : projects
  ModelEntity ||--o{ ModelInitialized : emits
  ModelEntity ||--o{ MetricUpdated : emits
  SubmissionQueue ||--o{ ReviewSubmitted : emits
  ReviewSubmissionConsumer }o--|| SubmissionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ReviewPlannerAgent` | `application/ReviewPlannerAgent.java` |
| `TranscriptReaderAgent` | `application/TranscriptReaderAgent.java` |
| `FilingReaderAgent` | `application/FilingReaderAgent.java` |
| `ModelUpdaterAgent` | `application/ModelUpdaterAgent.java` |
| `ThesisFlagAgent` | `application/ThesisFlagAgent.java` |
| `ReviewWorkflow` | `application/ReviewWorkflow.java` |
| `ReviewEntity` | `application/ReviewEntity.java` (state in `domain/Review.java`, events in `domain/ReviewEvent.java`) |
| `ModelEntity` | `application/ModelEntity.java` |
| `SubmissionQueue` | `application/SubmissionQueue.java` |
| `ReviewView` | `application/ReviewView.java` |
| `ReviewSubmissionConsumer` | `application/ReviewSubmissionConsumer.java` |
| `EarningsSimulator` | `application/EarningsSimulator.java` |
| `StaleReviewMonitor` | `application/StaleReviewMonitor.java` |
| `FlagGuardrail` | `application/FlagGuardrail.java` |
| `EvidenceQualityEvaluator` | `application/EvidenceQualityEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `SpecialistTasks` | `application/SpecialistTasks.java` |
| `ReviewEndpoint` | `api/ReviewEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Mermaid theme overrides (Lesson 24)

The generated `static-resources/index.html` includes the Akka theme variables AND CSS overrides — `transitionLabelColor #cccccc`, edge-label `foreignObject { overflow: visible }`, and explicit text colours on state-diagram nodes. Without these the state machine above renders unreadable.

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `finalizeStep` 60 s, `completeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(ReviewWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` becomes `Fail`.
- **Failure budget:** the planner may emit `Continue` on the same `(specialist, focus)` at most three times; a fourth becomes `Fail`.
- **Guardrail revise opportunity:** if every candidate flag is rejected by the guardrail, the planner is given one — and only one — opportunity to revise the candidate list. A second rejection sequence ends the review with empty `acceptedFlags`.
- **Idempotency:** `ReviewEndpoint.submit` deduplicates `(ticker, period, submittedBy)` over a 30 s window.
- **Stale detection:** `StaleReviewMonitor` ticks every 60 s; reviews in non-terminal states for > 5 minutes are marked `STALE`. The workflow's `decideStep` checks the entity status and exits if it reads `STALE`.
- **Model write fence:** `ModelEntity.updateMetric` runs only inside `applyModelStep`, after the guardrail and eval have cleared. A failed review never touches the model.
- **Determinism:** `FlagGuardrail.vet` and `EvidenceQualityEvaluator.score` are pure functions of the flag plus the ledger; the same input always yields the same verdict and score, which keeps the resulting `FlagAccepted` / `FlagRejected` / `FlagScored` events replayable.
