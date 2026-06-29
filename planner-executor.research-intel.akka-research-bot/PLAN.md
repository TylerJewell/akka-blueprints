# PLAN — akka-research-bot

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

  Planner[PlannerAgent]:::agent
  Searcher[SearcherAgent]:::agent
  Writer[WriterAgent]:::agent

  WF[ResearchWorkflow]:::wf
  Job[ResearchJobEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[JobQueue]:::ese
  View[ResearchJobView]:::view
  Consumer[JobRequestConsumer]:::cons
  Sim[JobSimulator]:::ta
  Stale[StaleJobMonitor]:::ta
  API[JobEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / ASSESS| Planner
  WF -->|SEARCH| Searcher
  WF -->|DRAFT_REPORT / REVISE_REPORT| Writer
  WF -->|emit events| Job
  WF -->|poll| Ctrl
  Job -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| Job
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as JobEndpoint
  participant Q as JobQueue
  participant C as JobRequestConsumer
  participant W as ResearchWorkflow
  participant P as PlannerAgent
  participant S as SearcherAgent
  participant Wr as WriterAgent
  participant E as ResearchJobEntity
  participant CTL as SystemControlEntity
  participant V as ResearchJobView

  U->>API: POST /api/jobs {question}
  API->>Q: append JobSubmitted
  API-->>U: 202 {jobId}
  Q->>C: JobSubmitted
  C->>W: start({jobId, question})
  W->>E: emit JobCreated (PLANNING)
  W->>P: PLAN(question)
  P-->>W: ResearchPlan
  W->>E: emit JobPlanned, status SEARCHING
  loop for each SearchQuery in plan
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>W: QueryGuardrail.vet(query)
    W->>S: SEARCH(query)
    S-->>W: SearchResult
    W->>W: SecretScrubber.scrub(content)
    W->>E: emit ResultRecorded (ResultEntry)
  end
  W->>P: ASSESS(results)
  P-->>W: Sufficient
  W->>E: emit WritingStarted, status WRITING
  W->>Wr: DRAFT_REPORT(results)
  Wr-->>W: ResearchReport
  W->>W: ReportGuardrail.inspect(report)
  W->>E: emit JobCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ResearchJobEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> SEARCHING: JobPlanned
  SEARCHING --> SEARCHING: ResultRecorded / QueryBlocked / PlanRevised
  SEARCHING --> WRITING: WritingStarted
  WRITING --> WRITING: ReportBlocked
  WRITING --> COMPLETED: JobCompleted
  SEARCHING --> FAILED: JobFailed
  WRITING --> FAILED: JobFailed
  SEARCHING --> HALTED: JobHaltedOperator
  SEARCHING --> STALE: JobFailedTimeout
  STALE --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  ResearchJobEntity ||--o{ JobPlanned : emits
  ResearchJobEntity ||--o{ QueryDispatched : emits
  ResearchJobEntity ||--o{ QueryBlocked : emits
  ResearchJobEntity ||--o{ ResultRecorded : emits
  ResearchJobEntity ||--o{ PlanRevised : emits
  ResearchJobEntity ||--o{ WritingStarted : emits
  ResearchJobEntity ||--o{ ReportBlocked : emits
  ResearchJobEntity ||--o{ JobCompleted : emits
  ResearchJobEntity ||--o{ JobFailed : emits
  ResearchJobEntity ||--o{ JobHaltedOperator : emits
  ResearchJobEntity ||--o{ JobFailedTimeout : emits
  ResearchJobView }o--|| ResearchJobEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  JobQueue ||--o{ JobSubmitted : emits
  JobRequestConsumer }o--|| JobQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `SearcherAgent` | `application/SearcherAgent.java` |
| `WriterAgent` | `application/WriterAgent.java` |
| `ResearchWorkflow` | `application/ResearchWorkflow.java` |
| `ResearchJobEntity` | `application/ResearchJobEntity.java` (state in `domain/ResearchJob.java`, events in `domain/JobEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `JobQueue` | `application/JobQueue.java` |
| `ResearchJobView` | `application/ResearchJobView.java` |
| `JobRequestConsumer` | `application/JobRequestConsumer.java` |
| `JobSimulator` | `application/JobSimulator.java` |
| `StaleJobMonitor` | `application/StaleJobMonitor.java` |
| `QueryGuardrail` | `application/QueryGuardrail.java` |
| `ReportGuardrail` | `application/ReportGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `SearcherTasks` | `application/SearcherTasks.java` |
| `WriterTasks` | `application/WriterTasks.java` |
| `JobEndpoint` | `api/JobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `searchStep` 90 s (one query per step; parallel fan-out is modelled as sequential steps over the query list), `assessStep` 45 s, `writeStep` 120 s (writer may produce a multi-section report), `reportGuardrailStep` 30 s. Default recovery: `maxRetries(2).failoverTo(ResearchWorkflow::error)`.
- **Replan budget:** the planner may return `NeedsMore` at most twice consecutively; a third consecutive `NeedsMore` is treated as `Fail`.
- **Report revision budget:** the writer may receive `REVISE_REPORT` at most twice; a third rejection by the report guardrail becomes `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `searchStep` lets the in-flight search finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `JobEndpoint.submit` uses `(question, requestedBy)` over a 10 s window to dedupe `POST /api/jobs`.
- **Stale detection:** `StaleJobMonitor` ticks every 30 s; tasks `SEARCHING` for > 5 minutes are marked `STALE`. The workflow's `checkHaltStep` checks the entity's status and exits if it reads `STALE`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure; it never inspects external state. The same input always yields the same scrubbed output, keeping `ResultEntry` events deterministic and replayable.
