# PLAN — Economic Research Agent

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  AE[AnalysisEndpoint<br/>HttpEndpoint]:::ep
  APE[AppEndpoint<br/>HttpEndpoint]:::ep
  RQ[RequestQueue<br/>EventSourcedEntity]:::ese
  MRC[MarketRequestConsumer<br/>Consumer]:::con
  WF[AnalysisWorkflow<br/>Workflow]:::wf
  EC[EconomicsCoordinator<br/>AutonomousAgent]:::ag
  DC[DataCollector<br/>AutonomousAgent]:::ag
  ECO[Economist<br/>AutonomousAgent]:::ag
  RE[AnalysisReportEntity<br/>EventSourcedEntity]:::ese
  VW[AnalysisView<br/>View]:::vw
  SIM[RequestSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  AE -->|POST /analysis| RQ
  SIM -.->|every 60s| RQ
  RQ -.->|QuestionSubmitted| MRC
  MRC -->|start workflow| WF
  WF -->|FRAME| EC
  WF -->|COLLECT| DC
  WF -->|INTERPRET| ECO
  WF -->|SYNTHESISE| EC
  WF -->|commands| RE
  RE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| RE
  AE -->|getAllReports / SSE| VW
  APE --> STATIC[static-resources]:::static

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
  participant AE as AnalysisEndpoint
  participant RQ as RequestQueue
  participant WF as AnalysisWorkflow
  participant EC as EconomicsCoordinator
  participant DC as DataCollector
  participant ECO as Economist
  participant RE as AnalysisReportEntity

  U->>AE: POST /api/analysis {question}
  AE->>RQ: enqueueQuestion
  RQ-->>WF: MarketRequestConsumer starts workflow
  WF->>RE: createReport (FRAMING)
  WF->>EC: FRAME -> ResearchPlan
  WF->>RE: status IN_PROGRESS
  par parallel fan-out
    WF->>DC: COLLECT -> IndicatorBundle
  and
    WF->>ECO: INTERPRET -> EconomicInterpretation
  end
  Note over WF: join; if either step times out (60s) -> degradeStep
  WF->>EC: SYNTHESISE(indicators, interpretation) -> SynthesisedReport
  WF->>WF: guardrailStep vets the report
  alt guardrail passes
    WF->>RE: publishReport (PUBLISHED)
  else guardrail fails
    WF->>RE: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> FRAMING
  FRAMING --> IN_PROGRESS: ResearchPlan ready
  IN_PROGRESS --> PUBLISHED: synthesise + guardrail pass
  IN_PROGRESS --> DEGRADED: a worker timed out
  IN_PROGRESS --> BLOCKED: guardrail fail
  DEGRADED --> [*]
  BLOCKED --> [*]
  PUBLISHED --> PUBLISHED: ReportEvalScored
  PUBLISHED --> [*]
```

## Entity model

```mermaid
erDiagram
  ANALYSIS_REPORT ||--o{ INDICATOR_BUNDLE : has
  ANALYSIS_REPORT ||--o{ ECONOMIC_INTERPRETATION : has
  ANALYSIS_REPORT ||--o| SYNTHESISED_REPORT : produces
  REQUEST_QUEUE ||--|| ANALYSIS_REPORT : seeds
  ANALYSIS_REPORT {
    string reportId
    string question
    enum status
    int evalScore
    instant createdAt
  }
  REQUEST_QUEUE {
    string reportId
    string question
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `EconomicsCoordinator` | AutonomousAgent | `application/EconomicsCoordinator.java` |
| `DataCollector` | AutonomousAgent | `application/DataCollector.java` |
| `Economist` | AutonomousAgent | `application/Economist.java` |
| `EconomicsTasks` | Task constants | `application/EconomicsTasks.java` |
| `AnalysisWorkflow` | Workflow | `application/AnalysisWorkflow.java` |
| `AnalysisReportEntity` | EventSourcedEntity | `domain/AnalysisReportEntity.java` |
| `RequestQueue` | EventSourcedEntity | `domain/RequestQueue.java` |
| `AnalysisView` | View | `application/AnalysisView.java` |
| `MarketRequestConsumer` | Consumer | `application/MarketRequestConsumer.java` |
| `RequestSimulator` | TimedAction | `application/RequestSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `AnalysisEndpoint` | HttpEndpoint | `api/AnalysisEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `collectStep` and `interpretStep` get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `collectStep` and `interpretStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Idempotency:** the workflow id is the `reportId`. Re-delivery of the same `QuestionSubmitted` event resolves to the same workflow instance — no duplicate report.
- **Degrade path (compensation):** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output exists and ends with `ReportDegraded`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `AnalysisView.getAllReports` (no enum WHERE clause) and filters client-side for the oldest `PUBLISHED` report lacking an `evalScore`.
