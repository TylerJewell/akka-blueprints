# PLAN — research-agent

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

  Planner[ResearchPlannerAgent]:::agent
  Web[WebSearchAgent]:::agent
  Doc[DocumentSearchAgent]:::agent

  WF[ResearchWorkflow]:::wf
  Job[ResearchJobEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[QueryQueue]:::ese
  View[ResearchJobView]:::view
  Consumer[QueryRequestConsumer]:::cons
  Sim[QuerySimulator]:::ta
  Stale[StaleJobMonitor]:::ta
  API[ResearchEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_SEARCH / DECIDE_NEXT / COMPOSE_REPORT| Planner
  WF -->|WEB_SEARCH| Web
  WF -->|DOC_SEARCH| Doc
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
  participant API as ResearchEndpoint
  participant Q as QueryQueue
  participant C as QueryRequestConsumer
  participant W as ResearchWorkflow
  participant P as ResearchPlannerAgent
  participant S as Searcher (Web/Document)
  participant E as ResearchJobEntity
  participant CTL as SystemControlEntity
  participant V as ResearchJobView

  U->>API: POST /api/jobs {query}
  API->>Q: append QuerySubmitted
  API-->>U: 202 {jobId}
  Q->>C: QuerySubmitted
  C->>W: start({jobId, query})
  W->>E: emit JobCreated (PLANNING)
  W->>P: PLAN_SEARCH(query)
  P-->>W: SearchLedger
  W->>E: emit JobPlanned, status SEARCHING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE_NEXT(ledgers)
    P-->>W: Continue(SearchDecision)
    W->>S: runSingleTask(query)
    S-->>W: SearchResult
    W->>W: CitationEvaluator.evaluate(result)
    W->>E: emit FindingRecorded (FindingEntry)
  end
  W->>P: COMPOSE_REPORT
  P-->>W: ResearchReport
  W->>E: emit JobCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ResearchJobEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> SEARCHING: JobPlanned
  SEARCHING --> SEARCHING: FindingRecorded / StepRetried / LedgerRevised
  SEARCHING --> COMPLETED: JobCompleted
  SEARCHING --> FAILED: JobFailed
  SEARCHING --> HALTED: JobHaltedOperator
  SEARCHING --> STUCK: JobFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  ResearchJobEntity ||--o{ JobPlanned : emits
  ResearchJobEntity ||--o{ StepDispatched : emits
  ResearchJobEntity ||--o{ StepRetried : emits
  ResearchJobEntity ||--o{ FindingRecorded : emits
  ResearchJobEntity ||--o{ LedgerRevised : emits
  ResearchJobEntity ||--o{ JobCompleted : emits
  ResearchJobEntity ||--o{ JobFailed : emits
  ResearchJobEntity ||--o{ JobHaltedOperator : emits
  ResearchJobEntity ||--o{ JobFailedTimeout : emits
  ResearchJobView }o--|| ResearchJobEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  QueryQueue ||--o{ QuerySubmitted : emits
  QueryRequestConsumer }o--|| QueryQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ResearchPlannerAgent` | `application/ResearchPlannerAgent.java` |
| `WebSearchAgent` | `application/WebSearchAgent.java` |
| `DocumentSearchAgent` | `application/DocumentSearchAgent.java` |
| `ResearchWorkflow` | `application/ResearchWorkflow.java` |
| `ResearchJobEntity` | `application/ResearchJobEntity.java` (state in `domain/ResearchJob.java`, events in `domain/JobEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `QueryQueue` | `application/QueryQueue.java` |
| `ResearchJobView` | `application/ResearchJobView.java` |
| `QueryRequestConsumer` | `application/QueryRequestConsumer.java` |
| `QuerySimulator` | `application/QuerySimulator.java` |
| `StaleJobMonitor` | `application/StaleJobMonitor.java` |
| `CitationEvaluator` | `application/CitationEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `SearcherTasks` | `application/SearcherTasks.java` |
| `ResearchEndpoint` | `api/ResearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any searcher call), `citationEvalStep` 15 s, `decideStep` 45 s, `composeReportStep` 60 s. Default recovery: `maxRetries(2).failoverTo(ResearchWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the planner may emit `Continue` on the same `(searcher, query)` at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchStep` lets the in-flight search step finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `ResearchEndpoint.submit` uses `(query, requestedBy)` over a 10 s window to deduplicate `POST /api/jobs`.
- **Stale detection:** `StaleJobMonitor` ticks every 30 s; `JobFailedTimeout` is non-fatal to other jobs. The workflow's `decideStep` checks the entity's status and exits if it reads `STUCK`.
- **Citation evaluator determinism:** `CitationEvaluator.evaluate` is pure; it never inspects external state. The same content always yields the same verdict and confidence, keeping `FindingEntry` events deterministic and replayable.
