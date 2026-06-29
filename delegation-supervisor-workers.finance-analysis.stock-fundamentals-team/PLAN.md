# PLAN — AI Team for Fundamental Stock Analysis

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  AE[AnalysisEndpoint<br/>HttpEndpoint]:::ep
  APE[AppEndpoint<br/>HttpEndpoint]:::ep
  TQ[TickerQueue<br/>EventSourcedEntity]:::ese
  TC[TickerRequestConsumer<br/>Consumer]:::con
  WF[AnalysisWorkflow<br/>Workflow]:::wf
  CO[AnalysisCoordinator<br/>AutonomousAgent]:::ag
  FA[FinancialsAgent<br/>AutonomousAgent]:::ag
  NA[NewsAgent<br/>AutonomousAgent]:::ag
  RA[RatioAgent<br/>AutonomousAgent]:::ag
  SE[StockReportEntity<br/>EventSourcedEntity]:::ese
  VW[StockReportView<br/>View]:::vw
  SIM[TickerSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  AE -->|POST /analysis| TQ
  SIM -.->|every 90s| TQ
  TQ -.->|TickerSubmitted| TC
  TC -->|start workflow| WF
  WF -->|DECOMPOSE| CO
  WF -->|EXTRACT_FINANCIALS| FA
  WF -->|GATHER_NEWS| NA
  WF -->|COMPUTE_RATIOS| RA
  WF -->|SYNTHESISE| CO
  WF -->|commands| SE
  SE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| SE
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
  participant TQ as TickerQueue
  participant WF as AnalysisWorkflow
  participant CO as AnalysisCoordinator
  participant FA as FinancialsAgent
  participant NA as NewsAgent
  participant RA as RatioAgent
  participant SE as StockReportEntity

  U->>AE: POST /api/analysis {ticker}
  AE->>TQ: enqueueTicker
  TQ-->>WF: TickerRequestConsumer starts workflow
  WF->>SE: createReport (QUEUED)
  WF->>CO: DECOMPOSE -> AnalysisSubtasks
  WF->>SE: status ANALYSING
  par parallel fan-out
    WF->>FA: EXTRACT_FINANCIALS -> FinancialMetrics
  and
    WF->>NA: GATHER_NEWS -> NewsSummary
  and
    WF->>RA: COMPUTE_RATIOS -> RatioSet
  end
  Note over WF: join; if any step times out (60s) -> degradeStep
  WF->>CO: SYNTHESISE(financials, news, ratios) -> StockRecommendation
  WF->>WF: guardrailStep vets for direct advice
  WF->>WF: sanitizerStep strips price-sensitive text
  alt both controls pass
    WF->>SE: complete (COMPLETE)
  else guardrail or sanitizer fails
    WF->>SE: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> ANALYSING: AnalysisSubtasks ready
  ANALYSING --> COMPLETE: synthesise + guardrail + sanitizer pass
  ANALYSING --> DEGRADED: a worker timed out
  ANALYSING --> BLOCKED: guardrail or sanitizer fail
  DEGRADED --> [*]
  BLOCKED --> [*]
  COMPLETE --> COMPLETE: ReportEvalScored
  COMPLETE --> [*]
```

## Entity model

```mermaid
erDiagram
  STOCK_REPORT ||--o| FINANCIAL_METRICS : has
  STOCK_REPORT ||--o| NEWS_SUMMARY : has
  STOCK_REPORT ||--o| RATIO_SET : has
  STOCK_REPORT ||--o| STOCK_RECOMMENDATION : produces
  TICKER_QUEUE ||--|| STOCK_REPORT : seeds
  STOCK_REPORT {
    string reportId
    string ticker
    enum status
    enum stance
    int evalScore
    instant createdAt
  }
  TICKER_QUEUE {
    string reportId
    string ticker
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `AnalysisCoordinator` | AutonomousAgent | `application/AnalysisCoordinator.java` |
| `FinancialsAgent` | AutonomousAgent | `application/FinancialsAgent.java` |
| `NewsAgent` | AutonomousAgent | `application/NewsAgent.java` |
| `RatioAgent` | AutonomousAgent | `application/RatioAgent.java` |
| `AnalysisTasks` | Task constants | `application/AnalysisTasks.java` |
| `AnalysisWorkflow` | Workflow | `application/AnalysisWorkflow.java` |
| `StockReportEntity` | EventSourcedEntity | `domain/StockReportEntity.java` |
| `TickerQueue` | EventSourcedEntity | `domain/TickerQueue.java` |
| `StockReportView` | View | `application/StockReportView.java` |
| `TickerRequestConsumer` | Consumer | `application/TickerRequestConsumer.java` |
| `TickerSimulator` | TimedAction | `application/TickerSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `AnalysisEndpoint` | HttpEndpoint | `api/AnalysisEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `financialsStep`, `newsStep`, and `ratiosStep` each get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `financialsStep`, `newsStep`, and `ratiosStep` run concurrently via `CompletionStage.allOf`, not three sequential step calls.
- **Idempotency:** the workflow id is the `reportId`. Re-delivery of the same `TickerSubmitted` event resolves to the same workflow instance — no duplicate report.
- **Degrade path (compensation):** if any worker times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial results exist and ends with `ReportDegraded`. No infinite retry.
- **Three-way join:** the join step collects whichever of `FinancialMetrics`, `NewsSummary`, and `RatioSet` completed before passing them to the Coordinator. Fields for missing workers are empty `Optional`s, and the Coordinator's prompt instructs it to note missing inputs in the summary.
- **Eval sampling:** `EvalSampler` reads `StockReportView.getAllReports` (no enum WHERE clause) and filters client-side for the oldest `COMPLETE` report lacking an `evalScore`.
