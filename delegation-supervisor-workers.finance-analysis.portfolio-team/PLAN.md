# PLAN — Portfolio Assistant (Multi-Agent)

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  PE[PortfolioEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  HQ[HoldingsQueue<br/>EventSourcedEntity]:::ese
  HC[HoldingsConsumer<br/>Consumer]:::con
  WF[PortfolioWorkflow<br/>Workflow]:::wf
  CO[PortfolioCoordinator<br/>AutonomousAgent]:::ag
  HA[HoldingsAnalyst<br/>AutonomousAgent]:::ag
  MC[MarketContextAgent<br/>AutonomousAgent]:::ag
  RE[PortfolioReportEntity<br/>EventSourcedEntity]:::ese
  VW[PortfolioView<br/>View]:::vw
  SIM[HoldingsSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  PE -->|POST /portfolio| HQ
  SIM -.->|every 60s| HQ
  HQ -.->|HoldingsSubmitted| HC
  HC -->|start workflow| WF
  WF -->|sanitizeStep| WF
  WF -->|PLAN| CO
  WF -->|ASSESS| HA
  WF -->|GATHER_CONTEXT| MC
  WF -->|CONSOLIDATE| CO
  WF -->|commands| RE
  RE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| RE
  PE -->|getAllReports / SSE| VW
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
  participant PE as PortfolioEndpoint
  participant HQ as HoldingsQueue
  participant WF as PortfolioWorkflow
  participant CO as PortfolioCoordinator
  participant HA as HoldingsAnalyst
  participant MC as MarketContextAgent
  participant RE as PortfolioReportEntity

  U->>PE: POST /api/portfolio {portfolioId, sector, holdings}
  PE->>WF: sanitizeStep (sector check)
  alt sector prohibited
    WF->>RE: block (BLOCKED, no agent work)
  else sector valid
    PE->>HQ: enqueueHoldings
    HQ-->>WF: HoldingsConsumer starts workflow
    WF->>RE: createReport (PLANNING)
    WF->>CO: PLAN -> AnalysisPlan
    WF->>RE: status IN_PROGRESS
    par parallel fan-out
      WF->>HA: ASSESS -> PositionAssessment
    and
      WF->>MC: GATHER_CONTEXT -> MarketContext
    end
    Note over WF: join; if either step times out (60s) -> degradeStep
    WF->>CO: CONSOLIDATE(assessment, context) -> ConsolidatedReport
    WF->>WF: guardrailStep vets the report
    alt guardrail passes
      WF->>RE: consolidate (CONSOLIDATED)
    else guardrail fails
      WF->>RE: block (BLOCKED)
    end
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> IN_PROGRESS: AnalysisPlan ready
  IN_PROGRESS --> CONSOLIDATED: consolidate + guardrail pass
  IN_PROGRESS --> DEGRADED: a worker timed out
  IN_PROGRESS --> BLOCKED: guardrail fail
  PLANNING --> BLOCKED: sector prohibited
  DEGRADED --> [*]
  BLOCKED --> [*]
  CONSOLIDATED --> CONSOLIDATED: ReportEvalScored
  CONSOLIDATED --> [*]
```

## Entity model

```mermaid
erDiagram
  PORTFOLIO_REPORT ||--o{ POSITION_ASSESSMENT : has
  PORTFOLIO_REPORT ||--o{ MARKET_CONTEXT : has
  PORTFOLIO_REPORT ||--o| CONSOLIDATED_REPORT : produces
  HOLDINGS_QUEUE ||--|| PORTFOLIO_REPORT : seeds
  PORTFOLIO_REPORT {
    string reportId
    string portfolioId
    string sector
    enum status
    int evalScore
    instant createdAt
  }
  HOLDINGS_QUEUE {
    string reportId
    string portfolioId
    string sector
    string submittedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `PortfolioCoordinator` | AutonomousAgent | `application/PortfolioCoordinator.java` |
| `HoldingsAnalyst` | AutonomousAgent | `application/HoldingsAnalyst.java` |
| `MarketContextAgent` | AutonomousAgent | `application/MarketContextAgent.java` |
| `PortfolioTasks` | Task constants | `application/PortfolioTasks.java` |
| `PortfolioWorkflow` | Workflow | `application/PortfolioWorkflow.java` |
| `PortfolioReportEntity` | EventSourcedEntity | `domain/PortfolioReportEntity.java` |
| `HoldingsQueue` | EventSourcedEntity | `domain/HoldingsQueue.java` |
| `PortfolioView` | View | `application/PortfolioView.java` |
| `HoldingsConsumer` | Consumer | `application/HoldingsConsumer.java` |
| `HoldingsSimulator` | TimedAction | `application/HoldingsSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `SectorRegistry` | Registry util | `domain/SectorRegistry.java` |
| `PortfolioEndpoint` | HttpEndpoint | `api/PortfolioEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `assessStep` and `contextStep` get 60s; `consolidateStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `assessStep` and `contextStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Sanitize-first gate:** `sanitizeStep` runs before any agent or entity work. A prohibited sector short-circuits the workflow immediately; no `HoldingsQueue` entry is written.
- **Idempotency:** the workflow id is the `reportId`. Re-delivery of the same `HoldingsSubmitted` event resolves to the same workflow instance — no duplicate report.
- **Degrade path (compensation):** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which consolidates from whichever partial output exists and ends with `ReportDegraded`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `PortfolioView.getAllReports` (no enum WHERE clause) and filters client-side for the oldest `CONSOLIDATED` report lacking an `evalScore`.
