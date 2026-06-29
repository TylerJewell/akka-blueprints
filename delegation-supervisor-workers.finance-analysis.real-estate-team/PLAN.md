# PLAN — Real Estate Investment Multi-Agent

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  DE[DealEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  DQ[DealQueue<br/>EventSourcedEntity]:::ese
  DC_CON[DealRequestConsumer<br/>Consumer]:::con
  WF[EvaluationWorkflow<br/>Workflow]:::wf
  CO[DealCoordinator<br/>AutonomousAgent]:::ag
  MS[MarketSpecialist<br/>AutonomousAgent]:::ag
  FS[FinanceSpecialist<br/>AutonomousAgent]:::ag
  ENT[DealEntity<br/>EventSourcedEntity]:::ese
  VW[DealView<br/>View]:::vw
  SIM[DealSimulator<br/>TimedAction]:::ta
  EV[RecommendationEvalSampler<br/>TimedAction]:::ta

  DE -->|POST /deals| DQ
  SIM -.->|every 60s| DQ
  DQ -.->|DealSubmitted| DC_CON
  DC_CON -->|start workflow| WF
  WF -->|sanitizerStep| WF
  WF -->|SCOPE| CO
  WF -->|RESEARCH_MARKET| MS
  WF -->|MODEL_FINANCE| FS
  WF -->|SYNTHESISE| CO
  WF -->|commands| ENT
  ENT -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordScore| ENT
  DE -->|getAllDeals / SSE| VW
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

Solid arrows: synchronous commands. Dashed arrows: event subscriptions or scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant DE as DealEndpoint
  participant DQ as DealQueue
  participant WF as EvaluationWorkflow
  participant CO as DealCoordinator
  participant MS as MarketSpecialist
  participant FS as FinanceSpecialist
  participant ENT as DealEntity

  U->>DE: POST /api/deals {address, propertyType, askingPriceDollars}
  DE->>DQ: submitDeal
  DQ-->>WF: DealRequestConsumer starts workflow
  WF->>WF: sanitizerStep (property type check)
  alt excluded property type
    WF->>ENT: rejectDeal (REJECTED)
  else passes sanitizer
    WF->>ENT: createDeal (SCOPING)
    WF->>CO: SCOPE -> EvaluationPlan
    WF->>ENT: status EVALUATING
    par parallel fan-out
      WF->>MS: RESEARCH_MARKET -> MarketReport
    and
      WF->>FS: MODEL_FINANCE -> FinancialModel
    end
    Note over WF: join; if either step times out (60s) -> degradeStep
    WF->>CO: SYNTHESISE(marketReport, financialModel) -> InvestmentRecommendation
    WF->>WF: guardrailStep vets the recommendation
    alt guardrail passes
      WF->>ENT: recommend (RECOMMENDED)
    else guardrail fails
      WF->>ENT: block (BLOCKED)
    end
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> SCOPING
  SCOPING --> REJECTED: excluded property type
  SCOPING --> EVALUATING: EvaluationPlan ready
  EVALUATING --> RECOMMENDED: synthesise + guardrail pass
  EVALUATING --> DEGRADED: a worker timed out
  EVALUATING --> BLOCKED: guardrail fail
  REJECTED --> [*]
  DEGRADED --> [*]
  BLOCKED --> [*]
  RECOMMENDED --> RECOMMENDED: RecommendationScored
  RECOMMENDED --> [*]
```

## Entity model

```mermaid
erDiagram
  DEAL_RECORD ||--o{ MARKET_REPORT : has
  DEAL_RECORD ||--o{ FINANCIAL_MODEL : has
  DEAL_RECORD ||--o| INVESTMENT_RECOMMENDATION : produces
  DEAL_QUEUE ||--|| DEAL_RECORD : seeds
  DEAL_RECORD {
    string dealId
    string address
    string propertyType
    double askingPriceDollars
    enum status
    int evalScore
    instant createdAt
  }
  DEAL_QUEUE {
    string dealId
    string address
    string propertyType
    double askingPriceDollars
    string submittedBy
    instant submittedAt
  }
  MARKET_REPORT {
    list comparables
    string neighborhoodTrend
    double estimatedMarketValueDollars
    instant gatheredAt
  }
  FINANCIAL_MODEL {
    double capRate
    double cashOnCashReturn
    double debtServiceCoverageRatio
    double netOperatingIncome
    instant modelledAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `DealCoordinator` | AutonomousAgent | `application/DealCoordinator.java` |
| `MarketSpecialist` | AutonomousAgent | `application/MarketSpecialist.java` |
| `FinanceSpecialist` | AutonomousAgent | `application/FinanceSpecialist.java` |
| `DealTasks` | Task constants | `application/DealTasks.java` |
| `EvaluationWorkflow` | Workflow | `application/EvaluationWorkflow.java` |
| `DealEntity` | EventSourcedEntity | `domain/DealEntity.java` |
| `DealQueue` | EventSourcedEntity | `domain/DealQueue.java` |
| `DealView` | View | `application/DealView.java` |
| `DealRequestConsumer` | Consumer | `application/DealRequestConsumer.java` |
| `DealSimulator` | TimedAction | `application/DealSimulator.java` |
| `RecommendationEvalSampler` | TimedAction | `application/RecommendationEvalSampler.java` |
| `DealEndpoint` | HttpEndpoint | `api/DealEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `marketStep` and `financeStep` get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `marketStep` and `financeStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Sanitizer-first:** `sanitizerStep` is a pure Java check on `DealSubmission.propertyType`. It runs before any `forAutonomousAgent` call. If matched, the workflow ends with `DealRejected` — no LLM token is consumed.
- **Idempotency:** the workflow id is the `dealId`. Re-delivery of the same `DealSubmitted` event resolves to the same workflow instance — no duplicate deal.
- **Degrade path (compensation):** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output exists and ends with `DealDegraded`. No infinite retry.
- **Eval sampling:** `RecommendationEvalSampler` reads `DealView.getAllDeals` (no enum WHERE clause) and filters client-side for the oldest `RECOMMENDED` deal lacking an `evalScore`.
