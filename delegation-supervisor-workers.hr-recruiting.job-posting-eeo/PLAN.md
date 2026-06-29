# PLAN — job-posting-eeo

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

  Lead[PostingLead]:::agent
  Culture[CultureAnalyst]:::agent
  Role[RoleAnalyst]:::agent

  WF[JobPostingWorkflow]:::wf
  Posting[JobPostingEntity]:::ese
  Queue[RequestQueue]:::ese
  View[JobPostingView]:::view
  Consumer[PostingRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  API[JobPostingEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue request| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|decompose / draft| Lead
  WF -->|analyze culture| Culture
  WF -->|analyze role| Role
  WF -->|emit events| Posting
  Posting -.->|projects| View
  API -->|query/SSE| View
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as JobPostingEndpoint
  participant Q as RequestQueue
  participant C as PostingRequestConsumer
  participant W as JobPostingWorkflow
  participant L as PostingLead
  participant Cu as CultureAnalyst
  participant R as RoleAnalyst
  participant E as JobPostingEntity
  participant V as JobPostingView

  U->>API: POST /api/postings {company, roleTitle}
  API->>Q: append RequestSubmitted
  API-->>U: 202 {postingId}
  Q->>C: RequestSubmitted
  C->>W: start({postingId, company, roleTitle})
  W->>E: emit PostingCreated (PLANNING)
  W->>L: decompose(company, roleTitle)
  L-->>W: WorkPlan{cultureBrief, roleBrief}
  W->>E: status ANALYZING
  par
    W->>Cu: analyzeCulture(cultureBrief)
    Cu-->>W: CultureProfile
  and
    W->>R: analyzeRole(roleBrief)
    R-->>W: RoleSpec
  end
  W->>L: draft(culture, role)
  L-->>W: DraftPosting (EEO guardrail ok)
  W->>E: emit PostingDrafted (DRAFTED)
  Note over W: sanitizer strips protected-class terms
  W->>E: emit PostingSanitized (SANITIZED)
  Note over W: documentation gate verifies EEO statement
  W->>E: emit PostingCleared (CLEARED)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `JobPostingEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> ANALYZING: PostingCreated done, workers dispatched
  ANALYZING --> DRAFTED: both workers returned; EEO guardrail OK
  ANALYZING --> BLOCKED: guardrail flagged discriminatory language
  ANALYZING --> DEGRADED: one worker timed out
  DRAFTED --> SANITIZED: protected-class terms removed
  SANITIZED --> CLEARED: documentation gate passed
  CLEARED --> [*]
  BLOCKED --> [*]
  DEGRADED --> [*]
```

## Entity model

```mermaid
erDiagram
  JobPostingEntity ||--o{ CultureAttached : emits
  JobPostingEntity ||--o{ RoleAttached : emits
  JobPostingEntity ||--o{ PostingDrafted : emits
  JobPostingEntity ||--o{ PostingBlocked : emits
  JobPostingEntity ||--o{ PostingDegraded : emits
  JobPostingEntity ||--o{ PostingSanitized : emits
  JobPostingEntity ||--o{ PostingCleared : emits
  JobPostingView }o--|| JobPostingEntity : projects
  RequestQueue ||--o{ RequestSubmitted : emits
  PostingRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PostingLead` | `application/PostingLead.java` |
| `CultureAnalyst` | `application/CultureAnalyst.java` |
| `RoleAnalyst` | `application/RoleAnalyst.java` |
| `JobPostingWorkflow` | `application/JobPostingWorkflow.java` |
| `JobPostingEntity` | `application/JobPostingEntity.java` (state in `domain/JobPosting.java`, events in `domain/JobPostingEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `JobPostingView` | `application/JobPostingView.java` |
| `PostingRequestConsumer` | `application/PostingRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `JobPostingEndpoint` | `api/JobPostingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `JobPostingTasks` | `application/JobPostingTasks.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** wrap `CultureAnalyst`, `RoleAnalyst`, and `PostingLead` draft calls in `WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(60))`. `WorkflowSettings` is nested in `Workflow` — no import. On timeout, transition to `DEGRADED` rather than failing the whole workflow.
- **Parallel fork:** `cultureStep` and `roleStep` use a CompletionStage zip. Both worker calls are initiated before either is awaited.
- **Idempotency:** `JobPostingEndpoint.submit` uses `(company, roleTitle, requestedBy)` over a 10 s window to deduplicate `POST /api/postings`.
- **Guardrail / sanitizer ordering:** the EEO guardrail runs inside `draftStep` (blocking, before the draft is persisted as `DRAFTED`); the sanitizer runs in the following `sanitizeStep` (transforms, then emits `PostingSanitized`). The documentation gate runs last in `complianceStep`.
- **No compensation needed:** all transitions are forward-only; terminal states (`CLEARED`, `BLOCKED`, `DEGRADED`) are absorbing.
