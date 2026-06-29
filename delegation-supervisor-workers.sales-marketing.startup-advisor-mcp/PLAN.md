# PLAN — Startup Advisor (MCP)

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  AE_EP[AdvisoryEndpoint<br/>HttpEndpoint]:::ep
  APP[AppEndpoint<br/>HttpEndpoint]:::ep
  BQ[BriefQueue<br/>EventSourcedEntity]:::ese
  BC[BriefConsumer<br/>Consumer]:::con
  WF[AdvisoryWorkflow<br/>Workflow]:::wf
  CO[AdvisorCoordinator<br/>AutonomousAgent]:::ag
  MR[MarketResearcher<br/>AutonomousAgent]:::ag
  GA[GrowthAnalyst<br/>AutonomousAgent]:::ag
  SE[AdvisorySessionEntity<br/>EventSourcedEntity]:::ese
  VW[AdvisoryView<br/>View]:::vw
  SIM[BriefSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  AE_EP -->|POST /advisory| BQ
  SIM -.->|every 60s| BQ
  BQ -.->|BriefSubmitted| BC
  BC -->|start workflow| WF
  WF -->|DECOMPOSE| CO
  WF -->|RESEARCH| MR
  WF -->|ANALYSE_CHANNELS| GA
  WF -->|SYNTHESISE| CO
  WF -->|commands| SE
  SE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| SE
  AE_EP -->|getAllSessions / SSE| VW
  APP --> STATIC[static-resources]:::static

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
  participant AE as AdvisoryEndpoint
  participant BQ as BriefQueue
  participant WF as AdvisoryWorkflow
  participant CO as AdvisorCoordinator
  participant MR as MarketResearcher
  participant GA as GrowthAnalyst
  participant SE as AdvisorySessionEntity

  U->>AE: POST /api/advisory {description}
  AE->>BQ: enqueueBrief
  BQ-->>WF: BriefConsumer starts workflow
  WF->>SE: createSession (SCOPING)
  WF->>CO: DECOMPOSE -> BriefDecomposition
  WF->>SE: status IN_PROGRESS
  par parallel fan-out
    WF->>MR: RESEARCH -> MarketSnapshot
    Note over MR: before-tool-call guardrail<br/>checks allow-list first
  and
    WF->>GA: ANALYSE_CHANNELS -> ChannelPlan
    Note over GA: before-tool-call guardrail<br/>checks allow-list first
  end
  Note over WF: join; if either step times out (60s) -> degradeStep
  WF->>CO: SYNTHESISE(snapshot, plan) -> GtmGuidance
  WF->>SE: synthesiseGuidance (ADVISED)
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> SCOPING
  SCOPING --> IN_PROGRESS: BriefDecomposition ready
  IN_PROGRESS --> ADVISED: guidance synthesised
  IN_PROGRESS --> DEGRADED: a worker timed out
  DEGRADED --> [*]
  ADVISED --> ADVISED: GuidanceEvalScored
  ADVISED --> [*]
```

## Entity model

```mermaid
erDiagram
  ADVISORY_SESSION ||--o| MARKET_SNAPSHOT : has
  ADVISORY_SESSION ||--o| CHANNEL_PLAN : has
  ADVISORY_SESSION ||--o| GTM_GUIDANCE : produces
  BRIEF_QUEUE ||--|| ADVISORY_SESSION : seeds
  ADVISORY_SESSION {
    string sessionId
    string description
    enum status
    int evalScore
    instant createdAt
  }
  BRIEF_QUEUE {
    string sessionId
    string description
    string submittedBy
    instant submittedAt
  }
  MARKET_SNAPSHOT {
    string tam
    list competitors
    instant researchedAt
  }
  CHANNEL_PLAN {
    string primaryChannel
    list channels
    instant analysedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `AdvisorCoordinator` | AutonomousAgent | `application/AdvisorCoordinator.java` |
| `MarketResearcher` | AutonomousAgent | `application/MarketResearcher.java` |
| `GrowthAnalyst` | AutonomousAgent | `application/GrowthAnalyst.java` |
| `AdvisoryTasks` | Task constants | `application/AdvisoryTasks.java` |
| `AdvisoryWorkflow` | Workflow | `application/AdvisoryWorkflow.java` |
| `AdvisorySessionEntity` | EventSourcedEntity | `domain/AdvisorySessionEntity.java` |
| `BriefQueue` | EventSourcedEntity | `domain/BriefQueue.java` |
| `AdvisoryView` | View | `application/AdvisoryView.java` |
| `BriefConsumer` | Consumer | `application/BriefConsumer.java` |
| `BriefSimulator` | TimedAction | `application/BriefSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `AdvisoryEndpoint` | HttpEndpoint | `api/AdvisoryEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `researchStep` and `analyseChannelsStep` get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `researchStep` and `analyseChannelsStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Before-tool-call guardrail:** the allow-list check runs synchronously before each MCP tool invocation inside `MarketResearcher` and `GrowthAnalyst`. A disallowed call is rejected immediately and surfaces a `failureReason`; no tool side-effects occur.
- **Idempotency:** the workflow id is the `sessionId`. Re-delivery of the same `BriefSubmitted` event resolves to the same workflow instance — no duplicate session.
- **Degrade path:** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output exists and ends with `SessionDegraded`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `AdvisoryView.getAllSessions` (no enum WHERE clause) and filters client-side for the oldest `ADVISED` session lacking an `evalScore`.
