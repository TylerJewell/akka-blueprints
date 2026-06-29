# PLAN — Retail AI Location Strategy

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  LE[LocationEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  SQ[SiteQueue<br/>EventSourcedEntity]:::ese
  SC[SiteRequestConsumer<br/>Consumer]:::con
  WF[LocationWorkflow<br/>Workflow]:::wf
  CO[LocationCoordinator<br/>AutonomousAgent]:::ag
  MA[MarketAnalyst<br/>AutonomousAgent]:::ag
  DA[DemographicsAnalyst<br/>AutonomousAgent]:::ag
  CE[CandidateSiteEntity<br/>EventSourcedEntity]:::ese
  VW[LocationView<br/>View]:::vw
  SIM[SiteSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  LE -->|POST /locations| SQ
  SIM -.->|every 60s| SQ
  SQ -.->|SiteSubmitted| SC
  SC -->|start workflow| WF
  WF -->|DECOMPOSE| CO
  WF -->|ASSESS_MARKET| MA
  WF -->|ASSESS_DEMOGRAPHICS| DA
  WF -->|SYNTHESISE| CO
  WF -->|commands| CE
  CE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| CE
  LE -->|getAllSites / SSE| VW
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
  participant LE as LocationEndpoint
  participant SQ as SiteQueue
  participant WF as LocationWorkflow
  participant CO as LocationCoordinator
  participant MA as MarketAnalyst
  participant DA as DemographicsAnalyst
  participant CE as CandidateSiteEntity

  U->>LE: POST /api/locations {address, city, region}
  LE->>SQ: enqueueSite
  SQ-->>WF: SiteRequestConsumer starts workflow
  WF->>CE: createSite (SCORING)
  WF->>CO: DECOMPOSE -> ScoringPlan
  WF->>CE: status IN_PROGRESS
  par parallel fan-out
    WF->>MA: ASSESS_MARKET -> MarketAssessment
  and
    WF->>DA: ASSESS_DEMOGRAPHICS -> DemographicAssessment
  end
  Note over WF: join; if either step times out (60s) -> degradeStep
  WF->>CO: SYNTHESISE(market, demographics) -> SiteRecommendation
  WF->>WF: guardrailStep vets the recommendation
  alt guardrail passes
    WF->>CE: recommend or notRecommend (RECOMMENDED / NOT_RECOMMENDED)
  else guardrail fails
    WF->>CE: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> SCORING
  SCORING --> IN_PROGRESS: ScoringPlan ready
  IN_PROGRESS --> RECOMMENDED: synthesise + verdict=RECOMMENDED + guardrail pass
  IN_PROGRESS --> NOT_RECOMMENDED: synthesise + verdict=NOT_RECOMMENDED + guardrail pass
  IN_PROGRESS --> DEGRADED: an analyst timed out
  IN_PROGRESS --> BLOCKED: guardrail fail
  DEGRADED --> [*]
  BLOCKED --> [*]
  RECOMMENDED --> RECOMMENDED: EvalScored
  NOT_RECOMMENDED --> NOT_RECOMMENDED: EvalScored
  RECOMMENDED --> [*]
  NOT_RECOMMENDED --> [*]
```

## Entity model

```mermaid
erDiagram
  CANDIDATE_SITE ||--o| MARKET_ASSESSMENT : has
  CANDIDATE_SITE ||--o| DEMOGRAPHIC_ASSESSMENT : has
  CANDIDATE_SITE ||--o| SITE_RECOMMENDATION : produces
  SITE_QUEUE ||--|| CANDIDATE_SITE : seeds
  CANDIDATE_SITE {
    string siteId
    string address
    string city
    string region
    enum status
    double score
    int evalScore
    instant createdAt
  }
  SITE_QUEUE {
    string siteId
    string address
    string city
    string region
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `LocationCoordinator` | AutonomousAgent | `application/LocationCoordinator.java` |
| `MarketAnalyst` | AutonomousAgent | `application/MarketAnalyst.java` |
| `DemographicsAnalyst` | AutonomousAgent | `application/DemographicsAnalyst.java` |
| `LocationTasks` | Task constants | `application/LocationTasks.java` |
| `LocationWorkflow` | Workflow | `application/LocationWorkflow.java` |
| `CandidateSiteEntity` | EventSourcedEntity | `domain/CandidateSiteEntity.java` |
| `SiteQueue` | EventSourcedEntity | `domain/SiteQueue.java` |
| `LocationView` | View | `application/LocationView.java` |
| `SiteRequestConsumer` | Consumer | `application/SiteRequestConsumer.java` |
| `SiteSimulator` | TimedAction | `application/SiteSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `LocationEndpoint` | HttpEndpoint | `api/LocationEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `assessMarketStep` and `assessDemographicsStep` get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `assessMarketStep` and `assessDemographicsStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Idempotency:** the workflow id is the `siteId`. Re-delivery of the same `SiteSubmitted` event resolves to the same workflow instance — no duplicate evaluation.
- **Degrade path (compensation):** if either analyst times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output exists and ends with `SiteDegraded`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `LocationView.getAllSites` (no enum WHERE clause — Lesson 2) and filters client-side for the oldest `RECOMMENDED` or `NOT_RECOMMENDED` site lacking an `evalScore`.
