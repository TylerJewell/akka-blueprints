# PLAN — Finance Assistant Swarm Agent

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  RE[ReportEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  RQ[RequestQueue<br/>EventSourcedEntity]:::ese
  RC[ReportRequestConsumer<br/>Consumer]:::con
  WF[ReportWorkflow<br/>Workflow]:::wf
  CO[ReportCoordinator<br/>AutonomousAgent]:::ag
  TR[TickerResolver<br/>AutonomousAgent]:::ag
  PA[PriceAnalyst<br/>AutonomousAgent]:::ag
  NA[NewsAnalyst<br/>AutonomousAgent]:::ag
  FA[FundamentalsAnalyst<br/>AutonomousAgent]:::ag
  SE[StockReportEntity<br/>EventSourcedEntity]:::ese
  VW[StockReportView<br/>View]:::vw
  SIM[TickerSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  RE -->|POST /reports| RQ
  SIM -.->|every 60s| RQ
  RQ -.->|QuerySubmitted| RC
  RC -->|start workflow| WF
  WF -->|PLAN_QUERY| CO
  WF -->|RESOLVE| TR
  WF -->|ANALYSE_PRICE| PA
  WF -->|ANALYSE_NEWS| NA
  WF -->|ANALYSE_FUNDAMENTALS| FA
  WF -->|INTEGRATE| CO
  WF -->|commands| SE
  SE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| SE
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
  participant RQ as RequestQueue
  participant WF as ReportWorkflow
  participant CO as ReportCoordinator
  participant TR as TickerResolver
  participant PA as PriceAnalyst
  participant NA as NewsAnalyst
  participant FA as FundamentalsAnalyst
  participant SE as StockReportEntity

  U->>RE: POST /api/reports {query}
  RE->>RQ: enqueueQuery
  RQ-->>WF: ReportRequestConsumer starts workflow
  WF->>SE: createReport (RESOLVING)
  WF->>CO: PLAN_QUERY -> QueryPlan
  WF->>TR: RESOLVE -> TickerResolution
  WF->>SE: resolveTicker (IN_PROGRESS)
  par parallel fan-out
    WF->>PA: ANALYSE_PRICE -> PriceSummary
  and
    WF->>NA: ANALYSE_NEWS -> NewsSummary
  and
    WF->>FA: ANALYSE_FUNDAMENTALS -> FundamentalsSnapshot
  end
  Note over WF: join; if any step times out (60s) -> partialStep
  WF->>CO: INTEGRATE(price, news, fundamentals) -> IntegratedReport
  WF->>WF: sanitizerStep scrubs compliance phrases
  WF->>WF: guardrailStep vets numeric accuracy
  alt both checks pass
    WF->>SE: publish (PUBLISHED)
  else either check fails
    WF->>SE: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RESOLVING
  RESOLVING --> IN_PROGRESS: TickerResolved
  IN_PROGRESS --> PUBLISHED: integrate + sanitizer + guardrail pass
  IN_PROGRESS --> PARTIAL: any worker timed out
  IN_PROGRESS --> BLOCKED: sanitizer or guardrail fail
  PARTIAL --> [*]
  BLOCKED --> [*]
  PUBLISHED --> PUBLISHED: EvalScored
  PUBLISHED --> [*]
```

## Entity model

```mermaid
erDiagram
  STOCK_REPORT ||--o| TICKER_RESOLUTION : resolves
  STOCK_REPORT ||--o{ PRICE_SUMMARY : has
  STOCK_REPORT ||--o{ NEWS_SUMMARY : has
  STOCK_REPORT ||--o{ FUNDAMENTALS_SNAPSHOT : has
  STOCK_REPORT ||--o| INTEGRATED_REPORT : produces
  REQUEST_QUEUE ||--|| STOCK_REPORT : seeds
  STOCK_REPORT {
    string reportId
    string query
    enum status
    int evalScore
    instant createdAt
  }
  REQUEST_QUEUE {
    string reportId
    string query
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `ReportCoordinator` | AutonomousAgent | `application/ReportCoordinator.java` |
| `TickerResolver` | AutonomousAgent | `application/TickerResolver.java` |
| `PriceAnalyst` | AutonomousAgent | `application/PriceAnalyst.java` |
| `NewsAnalyst` | AutonomousAgent | `application/NewsAnalyst.java` |
| `FundamentalsAnalyst` | AutonomousAgent | `application/FundamentalsAnalyst.java` |
| `FinanceTasks` | Task constants | `application/FinanceTasks.java` |
| `ReportWorkflow` | Workflow | `application/ReportWorkflow.java` |
| `StockReportEntity` | EventSourcedEntity | `domain/StockReportEntity.java` |
| `RequestQueue` | EventSourcedEntity | `domain/RequestQueue.java` |
| `StockReportView` | View | `application/StockReportView.java` |
| `ReportRequestConsumer` | Consumer | `application/ReportRequestConsumer.java` |
| `TickerSimulator` | TimedAction | `application/TickerSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `ReportEndpoint` | HttpEndpoint | `api/ReportEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `priceStep`, `newsStep`, and `fundamentalsStep` each get 60s; `integrateStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `priceStep`, `newsStep`, and `fundamentalsStep` run concurrently via `CompletionStage` zip, not three sequential step calls.
- **Idempotency:** the workflow id is the `reportId`. Re-delivery of the same `QuerySubmitted` event resolves to the same workflow instance — no duplicate report.
- **Degrade path (compensation):** if any worker times out, `defaultStepRecovery` routes to `partialStep`, which integrates from whichever partial outputs exist and ends with `ReportPartial`. No infinite retry.
- **Sanitizer before guardrail:** `sanitizerStep` runs first (deterministic phrase-scrub); `guardrailStep` runs second (LLM judge). Either failure → `BLOCKED`.
- **Eval sampling:** `EvalSampler` reads `StockReportView.getAllReports` (no enum WHERE clause) and filters client-side for the oldest `PUBLISHED` report lacking an `evalScore`.
