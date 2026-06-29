# PLAN — scored-loop-tailor

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

  Tailor[CvTailorAgent]:::agent
  Reviewer[CvReviewerAgent]:::agent

  WF[TailoringWorkflow]:::wf
  App[ApplicationEntity]:::ese
  Queue[SubmissionQueue]:::ese
  View[ApplicationsView]:::view
  Consumer[SubmissionConsumer]:::cons
  Sim[SubmissionSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[RecruitingEndpoint]:::ep
  AppEP[AppEndpoint]:::ep

  API -->|submit posting| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|tailor / revise| Tailor
  WF -->|review| Reviewer
  WF -->|emit events| App
  App -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| App
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as RecruitingEndpoint
  participant Q as SubmissionQueue
  participant C as SubmissionConsumer
  participant W as TailoringWorkflow
  participant T as CvTailorAgent
  participant R as CvReviewerAgent
  participant E as ApplicationEntity
  participant V as ApplicationsView

  U->>API: POST /api/applications {jobTitle, description}
  API->>Q: append JobPostingSubmitted
  API-->>U: 202 {applicationId}
  Q->>C: JobPostingSubmitted
  C->>W: start({applicationId, jobTitle, description, maxAttempts=4})
  W->>E: emit ApplicationCreated (TAILORING)

  W->>T: TAILOR_CV(jobTitle, description)
  T-->>W: CvDraft #1 (all sections present)
  W->>E: emit CvDrafted (n=1)
  Note over W: guardrailStep (deterministic section check)
  W->>E: emit SectionGuardrailVerdictRecorded (passed=true)
  W->>E: status REVIEWING
  W->>R: REVIEW_CV(CvDraft #1)
  R-->>W: CvReview{REVISE, score=3, 3 bullets}
  W->>E: emit CvReviewed (n=1, REVISE)

  W->>T: REVISE_CV(jobPosting, priorDraft, notes)
  T-->>W: CvDraft #2 (all sections, bullets addressed)
  W->>E: emit CvDrafted (n=2)
  W->>E: emit SectionGuardrailVerdictRecorded (passed=true)
  W->>R: REVIEW_CV(CvDraft #2)
  R-->>W: CvReview{APPROVE, score=5, rationale}
  W->>E: emit CvReviewed (n=2, APPROVE)
  W->>E: emit ApplicationApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ApplicationEntity`

```mermaid
stateDiagram-v2
  [*] --> TAILORING
  TAILORING --> REVIEWING: CvDrafted + guardrail passed
  TAILORING --> TAILORING: guardrail blocked, re-tailor
  REVIEWING --> TAILORING: Review = REVISE, attempts < max
  REVIEWING --> APPROVED: Review = APPROVE
  REVIEWING --> REJECTED_FINAL: Review = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  ApplicationEntity ||--o{ ApplicationCreated : emits
  ApplicationEntity ||--o{ CvDrafted : emits
  ApplicationEntity ||--o{ SectionGuardrailVerdictRecorded : emits
  ApplicationEntity ||--o{ CvReviewed : emits
  ApplicationEntity ||--o{ ApplicationApproved : emits
  ApplicationEntity ||--o{ ApplicationRejectedFinal : emits
  ApplicationEntity ||--o{ ReviewEvalRecorded : emits
  ApplicationsView }o--|| ApplicationEntity : projects
  SubmissionQueue ||--o{ JobPostingSubmitted : emits
  SubmissionConsumer }o--|| SubmissionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CvTailorAgent` | `application/CvTailorAgent.java` |
| `CvReviewerAgent` | `application/CvReviewerAgent.java` |
| `RecruitingTasks` | `application/RecruitingTasks.java` |
| `TailoringWorkflow` | `application/TailoringWorkflow.java` |
| `ApplicationEntity` | `application/ApplicationEntity.java` (state in `domain/Application.java`, events in `domain/ApplicationEvent.java`) |
| `SubmissionQueue` | `application/SubmissionQueue.java` |
| `ApplicationsView` | `application/ApplicationsView.java` |
| `SubmissionConsumer` | `application/SubmissionConsumer.java` |
| `SubmissionSimulator` | `application/SubmissionSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `RecruitingEndpoint` | `api/RecruitingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `tailorStep` and `reviewStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `RecruitingEndpoint.submit` uses `(jobTitle, candidateProfile)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(applicationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `scored-loop-tailor.tailoring.max-attempts` (default 4). The workflow checks the count BEFORE calling `tailorStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the highest-scoring draft and every review on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it checks `draft.sectionsPresent()` for the strings "Summary", "Experience", and "Skills", and either advances to `reviewStep` or returns to `tailorStep` with a structured feedback note. The structured feedback never becomes an LLM-generated review; it stays a deterministic `ReviewNotes` payload with a single bullet.
