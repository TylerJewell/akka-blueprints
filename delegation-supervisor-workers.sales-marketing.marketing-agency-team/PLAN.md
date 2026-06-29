# PLAN — Marketing Agency Team

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  CE[CampaignEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  RQ[RequestQueue<br/>EventSourcedEntity]:::ese
  CC[CampaignRequestConsumer<br/>Consumer]:::con
  WF[CampaignWorkflow<br/>Workflow]:::wf
  CD[CampaignDirector<br/>AutonomousAgent]:::ag
  WL[WebsiteLauncher<br/>AutonomousAgent]:::ag
  SA[StrategyAdvisor<br/>AutonomousAgent]:::ag
  PE[CampaignPlanEntity<br/>EventSourcedEntity]:::ese
  VW[CampaignView<br/>View]:::vw
  SIM[RequestSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  CE -->|POST /campaigns| RQ
  SIM -.->|every 60s| RQ
  RQ -.->|CampaignBriefSubmitted| CC
  CC -->|start workflow| WF
  WF -->|SCOPE| CD
  WF -->|LAUNCH| WL
  WF -->|STRATEGISE| SA
  WF -->|SYNTHESISE| CD
  WF -->|commands| PE
  PE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| PE
  CE -->|getAllPlans / SSE| VW
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
  participant CE as CampaignEndpoint
  participant RQ as RequestQueue
  participant WF as CampaignWorkflow
  participant CD as CampaignDirector
  participant WL as WebsiteLauncher
  participant SA as StrategyAdvisor
  participant PE as CampaignPlanEntity

  U->>CE: POST /api/campaigns {campaignName, objective}
  CE->>RQ: submitBrief
  RQ-->>WF: CampaignRequestConsumer starts workflow
  WF->>PE: createPlan (PLANNING)
  WF->>CD: SCOPE -> WorkScope
  WF->>PE: status IN_PROGRESS
  par parallel fan-out
    WF->>WL: LAUNCH -> LaunchBrief
  and
    WF->>SA: STRATEGISE -> StrategyFramework
  end
  Note over WF: join; if either step times out (60s) -> degradeStep
  WF->>CD: SYNTHESISE(launchBrief, strategyFramework) -> SynthesisedPlan
  WF->>WF: guardrailStep vets the plan
  alt guardrail passes
    WF->>PE: synthesise (SYNTHESISED)
  else guardrail fails
    WF->>PE: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> IN_PROGRESS: WorkScope ready
  IN_PROGRESS --> SYNTHESISED: synthesise + guardrail pass
  IN_PROGRESS --> DEGRADED: a worker timed out
  IN_PROGRESS --> BLOCKED: guardrail fail
  DEGRADED --> [*]
  BLOCKED --> [*]
  SYNTHESISED --> SYNTHESISED: PlanEvalScored
  SYNTHESISED --> [*]
```

## Entity model

```mermaid
erDiagram
  CAMPAIGN_PLAN ||--o{ LAUNCH_BRIEF : has
  CAMPAIGN_PLAN ||--o{ STRATEGY_FRAMEWORK : has
  CAMPAIGN_PLAN ||--o| SYNTHESISED_PLAN : produces
  REQUEST_QUEUE ||--|| CAMPAIGN_PLAN : seeds
  CAMPAIGN_PLAN {
    string planId
    string campaignName
    string objective
    enum status
    int evalScore
    instant createdAt
  }
  REQUEST_QUEUE {
    string planId
    string campaignName
    string objective
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `CampaignDirector` | AutonomousAgent | `application/CampaignDirector.java` |
| `WebsiteLauncher` | AutonomousAgent | `application/WebsiteLauncher.java` |
| `StrategyAdvisor` | AutonomousAgent | `application/StrategyAdvisor.java` |
| `MarketingTasks` | Task constants | `application/MarketingTasks.java` |
| `CampaignWorkflow` | Workflow | `application/CampaignWorkflow.java` |
| `CampaignPlanEntity` | EventSourcedEntity | `domain/CampaignPlanEntity.java` |
| `RequestQueue` | EventSourcedEntity | `domain/RequestQueue.java` |
| `CampaignView` | View | `application/CampaignView.java` |
| `CampaignRequestConsumer` | Consumer | `application/CampaignRequestConsumer.java` |
| `RequestSimulator` | TimedAction | `application/RequestSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `CampaignEndpoint` | HttpEndpoint | `api/CampaignEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `launchStep` and `strategiseStep` get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `launchStep` and `strategiseStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Idempotency:** the workflow id is the `planId`. Re-delivery of the same `CampaignBriefSubmitted` event resolves to the same workflow instance — no duplicate plan.
- **Degrade path (compensation):** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output exists and ends with `PlanDegraded`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `CampaignView.getAllPlans` (no enum WHERE clause) and filters client-side for the oldest `SYNTHESISED` plan lacking an `evalScore`.
