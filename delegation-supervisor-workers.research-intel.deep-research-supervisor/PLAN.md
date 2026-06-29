# PLAN — Deep Research Supervisor

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  RE[ReportEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  QQ[QuestionQueue<br/>EventSourcedEntity]:::ese
  QC[QuestionConsumer<br/>Consumer]:::con
  WF[DeepResearchWorkflow<br/>Workflow]:::wf
  SU[ResearchSupervisor<br/>AutonomousAgent]:::ag
  SW[SearchWorker<br/>AutonomousAgent]:::ag
  EW[ExtractionWorker<br/>AutonomousAgent]:::ag
  MW[SummaryWorker<br/>AutonomousAgent]:::ag
  RE2[ResearchReportEntity<br/>EventSourcedEntity]:::ese
  VW[ReportView<br/>View]:::vw
  SIM[QuestionSimulator<br/>TimedAction]:::ta
  EV[CitationEvalSampler<br/>TimedAction]:::ta

  RE -->|POST /reports| QQ
  SIM -.->|every 60s| QQ
  QQ -.->|QuestionSubmitted| QC
  QC -->|start workflow| WF
  WF -->|DECOMPOSE| SU
  WF -->|SEARCH per subquery| SW
  WF -->|EXTRACT per subquery| EW
  WF -->|SUMMARISE per subquery| MW
  WF -->|SYNTHESISE| SU
  WF -->|commands| RE2
  RE2 -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| RE2
  RE -->|getAllReports / SSE| VW
  AE --> STATIC[static-resources]:::static

  classDef ep fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ese fill:#141414,stroke:#F5C518,color:#fff;
  classDef vw fill:#141414,stroke:#3fb950,color:#fff;
  classDef wf fill:#141414,stroke:#ff5f57,color:#fff;
  classDef ag fill:#141414,stroke:#B388FF,color:#fff;
  classDef con fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ta fill:#141414,stroke:#F5C518,color:#fff;
  classDef static fill:#0A0A0A,stroke:#333,color:#aaa;
```

Solid arrows: synchronous commands. Dashed arrows: event subscriptions. Dotted arrows: scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant RE as ReportEndpoint
  participant QQ as QuestionQueue
  participant WF as DeepResearchWorkflow
  participant SU as ResearchSupervisor
  participant SW as SearchWorker
  participant EW as ExtractionWorker
  participant MW as SummaryWorker
  participant RRE as ResearchReportEntity

  U->>RE: POST /api/reports {question}
  RE->>QQ: enqueueQuestion
  QQ-->>WF: QuestionConsumer starts workflow
  WF->>RRE: createReport (PLANNING)
  WF->>SU: DECOMPOSE -> DecompositionPlan
  WF->>RRE: status IN_PROGRESS
  par per-subquery pipelines (parallel)
    WF->>SW: SEARCH subquery-1 -> PassageBundle
    WF->>EW: EXTRACT passages-1 -> ClaimsBundle
    WF->>MW: SUMMARISE claims-1 -> SubquerySummary
  and
    WF->>SW: SEARCH subquery-2 -> PassageBundle
    WF->>EW: EXTRACT passages-2 -> ClaimsBundle
    WF->>MW: SUMMARISE claims-2 -> SubquerySummary
  end
  Note over WF: join; if any searchStep times out (60s) -> partial path
  WF->>SU: SYNTHESISE(subquerySummaries) -> SynthesisedReport
  WF->>WF: guardrailStep vets the report
  alt guardrail passes
    WF->>RRE: synthesise (SYNTHESISED)
  else guardrail fails
    WF->>RRE: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> IN_PROGRESS: DecompositionPlan ready
  IN_PROGRESS --> SYNTHESISED: synthesise + guardrail pass
  IN_PROGRESS --> DEGRADED: one or more subqueries partial
  IN_PROGRESS --> BLOCKED: guardrail fail
  DEGRADED --> [*]
  BLOCKED --> [*]
  SYNTHESISED --> SYNTHESISED: ReportEvalScored
  SYNTHESISED --> [*]
```

## Entity model

```mermaid
erDiagram
  RESEARCH_REPORT ||--o{ SUBQUERY_SUMMARY : contains
  RESEARCH_REPORT ||--o| SYNTHESISED_REPORT : produces
  RESEARCH_REPORT ||--o{ CITATION : cites
  QUESTION_QUEUE ||--|| RESEARCH_REPORT : seeds
  RESEARCH_REPORT {
    string reportId
    string question
    enum status
    int evalScore
    instant createdAt
  }
  QUESTION_QUEUE {
    string reportId
    string question
    string requestedBy
    instant submittedAt
  }
  SUBQUERY_SUMMARY {
    string subqueryId
    string queryText
    string summary
    instant summarisedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `ResearchSupervisor` | AutonomousAgent | `application/ResearchSupervisor.java` |
| `SearchWorker` | AutonomousAgent | `application/SearchWorker.java` |
| `ExtractionWorker` | AutonomousAgent | `application/ExtractionWorker.java` |
| `SummaryWorker` | AutonomousAgent | `application/SummaryWorker.java` |
| `DeepResearchTasks` | Task constants | `application/DeepResearchTasks.java` |
| `DeepResearchWorkflow` | Workflow | `application/DeepResearchWorkflow.java` |
| `ResearchReportEntity` | EventSourcedEntity | `domain/ResearchReportEntity.java` |
| `QuestionQueue` | EventSourcedEntity | `domain/QuestionQueue.java` |
| `ReportView` | View | `application/ReportView.java` |
| `QuestionConsumer` | Consumer | `application/QuestionConsumer.java` |
| `QuestionSimulator` | TimedAction | `application/QuestionSimulator.java` |
| `CitationEvalSampler` | TimedAction | `application/CitationEvalSampler.java` |
| `ReportEndpoint` | HttpEndpoint | `api/ReportEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `searchStep` gets 60s; `extractStep` and `summariseStep` each get 45s; `synthesiseStep` gets 120s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** per-subquery pipelines run concurrently via `CompletionStage` allOf across all subqueries. Within each subquery the three steps run sequentially (search → extract → summarise) because each step depends on the previous step's output.
- **Idempotency:** the workflow id is the `reportId`. Re-delivery of the same `QuestionSubmitted` event resolves to the same workflow instance — no duplicate report.
- **Degrade path (compensation):** if any `searchStep` times out, that subquery is marked partial and the workflow continues with the remaining subqueries. After synthesis, `defaultStepRecovery` routes to `degradeStep` which ends with `ReportDegraded`. No infinite retry.
- **Eval sampling:** `CitationEvalSampler` reads `ReportView.getAllReports` (no enum WHERE clause) and filters client-side for the oldest `SYNTHESISED` report lacking an `evalScore`.
