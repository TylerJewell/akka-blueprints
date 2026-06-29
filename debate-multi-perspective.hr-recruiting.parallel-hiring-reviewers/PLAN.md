# PLAN — parallel-hiring-reviewers

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab. All four mermaid diagrams use the Akka theme palette; the state diagram carries the Lesson 24 CSS overrides so state names render white and edge labels are not clipped.

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

  Coordinator[PanelCoordinator]:::agent
  HR[HrReviewer]:::agent
  Manager[ManagerReviewer]:::agent
  Team[TeamReviewer]:::agent
  Judge[FairnessJudge]:::agent

  WF[ReviewWorkflow]:::wf
  Candidate[CandidateEntity]:::ese
  Queue[CandidateQueue]:::ese
  View[EvaluationView]:::view
  Consumer[CandidateConsumer]:::cons
  Sim[CandidateSimulator]:::ta
  Eval[FairnessSampler]:::ta
  API[ReviewEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue profile| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|plan / aggregate| Coordinator
  WF -->|review| HR
  WF -->|review| Manager
  WF -->|review| Team
  WF -->|emit events| Candidate
  Candidate -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 10m| Judge
  Eval -.->|every 10m| Candidate
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. The special-category sanitizer is a deterministic helper invoked inside `ReviewWorkflow.sanitizeStep` — it has no component box because it makes no Akka call of its own.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ReviewEndpoint
  participant Q as CandidateQueue
  participant C as CandidateConsumer
  participant W as ReviewWorkflow
  participant PC as PanelCoordinator
  participant HR as HrReviewer
  participant MG as ManagerReviewer
  participant TM as TeamReviewer
  participant E as CandidateEntity
  participant V as EvaluationView

  U->>API: POST /api/evaluations {candidateName, roleApplied, resumeText}
  API->>Q: append ProfileReceived
  API-->>U: 202 {evaluationId}
  Q->>C: ProfileReceived
  C->>W: start({evaluationId, candidateName, roleApplied, rawResumeText})
  W->>E: emit EvaluationCreated (INTAKE)
  Note over W: sanitizeStep redacts protected attributes from rawResumeText<br/>raw text never persisted
  W->>E: emit ProfileSanitized (EVALUATING)
  W->>PC: plan(redactedResumeText)
  PC-->>W: EvaluationPlan{hrFocus, managerFocus, teamFocus}
  par
    W->>HR: reviewHr(redactedResumeText, focus)
    HR-->>W: PerspectiveReview(HR)
  and
    W->>MG: reviewManager(redactedResumeText, focus)
    MG-->>W: PerspectiveReview(MANAGER)
  and
    W->>TM: reviewTeam(redactedResumeText, focus)
    TM-->>W: PerspectiveReview(TEAM)
  end
  W->>PC: aggregate(perspectiveReviews)
  PC-->>W: HiringRecommendation (guardrail vetted)
  W->>E: emit RecommendationDecided (DECIDED)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `CandidateEntity`

```mermaid
stateDiagram-v2
  [*] --> INTAKE
  INTAKE --> EVALUATING: ProfileSanitized (protected attrs redacted)
  EVALUATING --> DECIDED: all perspectives reviewed; guardrail OK
  EVALUATING --> DEGRADED: a reviewer timed out
  EVALUATING --> BLOCKED: guardrail rejected the recommendation
  DECIDED --> DECIDED: FairnessScored (no status change)
  DECIDED --> [*]
  DEGRADED --> [*]
  BLOCKED --> [*]
```

## Entity model

```mermaid
erDiagram
  CandidateEntity ||--o{ EvaluationCreated : emits
  CandidateEntity ||--o{ ProfileSanitized : emits
  CandidateEntity ||--o{ HrReviewAttached : emits
  CandidateEntity ||--o{ ManagerReviewAttached : emits
  CandidateEntity ||--o{ TeamReviewAttached : emits
  CandidateEntity ||--o{ RecommendationDecided : emits
  CandidateEntity ||--o{ EvaluationDegraded : emits
  CandidateEntity ||--o{ EvaluationBlocked : emits
  CandidateEntity ||--o{ FairnessScored : emits
  EvaluationView }o--|| CandidateEntity : projects
  CandidateQueue ||--o{ ProfileReceived : emits
  CandidateConsumer }o--|| CandidateQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PanelCoordinator` | `application/PanelCoordinator.java` |
| `HrReviewer` | `application/HrReviewer.java` |
| `ManagerReviewer` | `application/ManagerReviewer.java` |
| `TeamReviewer` | `application/TeamReviewer.java` |
| `FairnessJudge` | `application/FairnessJudge.java` |
| `ReviewTasks` | `application/ReviewTasks.java` |
| `SpecialCategorySanitizer` | `application/SpecialCategorySanitizer.java` |
| `ReviewWorkflow` | `application/ReviewWorkflow.java` |
| `CandidateEntity` | `application/CandidateEntity.java` (state in `domain/CandidateEvaluation.java`, events in `domain/EvaluationEvent.java`) |
| `CandidateQueue` | `application/CandidateQueue.java` |
| `EvaluationView` | `application/EvaluationView.java` |
| `CandidateConsumer` | `application/CandidateConsumer.java` |
| `CandidateSimulator` | `application/CandidateSimulator.java` |
| `FairnessSampler` | `application/FairnessSampler.java` |
| `ReviewEndpoint` | `api/ReviewEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **2 http-endpoint · 2 timed-action · 1 view · 1 workflow · 1 service-setup · 5 autonomous-agent · 1 consumer · 2 event-sourced-entity**.

## Concurrency notes

- **Workflow step timeouts:** wrap the three reviewer calls and the aggregate call in `WorkflowSettings.builder().stepTimeout(MyStep, Duration.ofSeconds(60))`. The default 5-second step timeout (Lesson 4) is far too short for LLM calls — without the override every reviewer step retries forever.
- **Parallel fork:** `hrStep`, `managerStep`, and `teamStep` use Akka's parallel-step idiom (CompletionStage zip). All three calls must be initiated before any is awaited; sequential calls would defeat the debate-multi-perspective pattern.
- **Degraded path:** a reviewer timeout transitions to aggregation from partial input rather than failing the whole workflow; `failureReason` names the missing reviewer; status is `DEGRADED`.
- **Sanitizer ordering:** `sanitizeStep` runs before `planStep`. The raw résumé text lives only in the workflow's transient start command and is never written to `CandidateEntity` — only `redactedResumeText` is persisted. This realises control S1.
- **Idempotency:** `ReviewEndpoint.submit` uses `(candidateName, roleApplied)` over a 10-second window as the idempotency key to avoid double-creation on client retry.
- **View indexing:** `EvaluationView` exposes one query, `getAllEvaluations`, with no `WHERE status` clause — Akka cannot auto-index the `EvaluationStatus` enum column (Lesson 2). Callers filter by status client-side.
- **Fairness sampling:** `FairnessSampler` selects the oldest `DECIDED` evaluation with no `fairnessScore`, one per tick. `FairnessScored` does not change status; it only populates the score and rationale.
- **emptyState:** `CandidateEntity.emptyState()` returns `CandidateEvaluation.initial("", "", "")` with placeholder identity values and never references `commandContext()` (Lesson 3).
