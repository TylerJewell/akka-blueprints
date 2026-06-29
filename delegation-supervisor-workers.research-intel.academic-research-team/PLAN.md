# PLAN — Academic Research Team

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  DE[DigestEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  QQ[QueryQueue<br/>EventSourcedEntity]:::ese
  QC[QueryRequestConsumer<br/>Consumer]:::con
  WF[ResearchWorkflow<br/>Workflow]:::wf
  LC[LiteratureCoordinator<br/>AutonomousAgent]:::ag
  PS[PaperScout<br/>AutonomousAgent]:::ag
  DA[DomainAnalyst<br/>AutonomousAgent]:::ag
  RDE[ResearchDigestEntity<br/>EventSourcedEntity]:::ese
  DV[DigestView<br/>View]:::vw
  SIM[QuerySimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  DE -->|POST /digests| QQ
  SIM -.->|every 60s| QQ
  QQ -.->|QuerySubmitted| QC
  QC -->|start workflow| WF
  WF -->|DECOMPOSE| LC
  WF -->|SCOUT| PS
  WF -->|ANALYSE| DA
  WF -->|SYNTHESISE| LC
  WF -->|commands| RDE
  RDE -.->|events| DV
  EV -.->|every 5m| DV
  EV -->|recordEval| RDE
  DE -->|getAllDigests / SSE| DV
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
  participant DE as DigestEndpoint
  participant QQ as QueryQueue
  participant WF as ResearchWorkflow
  participant LC as LiteratureCoordinator
  participant PS as PaperScout
  participant DA as DomainAnalyst
  participant RDE as ResearchDigestEntity

  U->>DE: POST /api/digests {query}
  DE->>QQ: enqueueQuery
  QQ-->>WF: QueryRequestConsumer starts workflow
  WF->>RDE: createDigest (QUEUED)
  WF->>LC: DECOMPOSE -> ScoutingPlan
  WF->>RDE: status SCANNING
  par parallel fan-out
    WF->>PS: SCOUT -> PublicationBundle
  and
    WF->>DA: ANALYSE -> TrendReport
  end
  Note over WF: join; if either step times out (60s) -> degradeStep
  WF->>LC: SYNTHESISE(publications, trends) -> SynthesisedDigest
  WF->>WF: guardrailStep vets citations
  alt guardrail passes
    WF->>RDE: synthesise (SYNTHESISED)
  else guardrail fails
    WF->>RDE: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> SCANNING: ScoutingPlan ready
  SCANNING --> SYNTHESISED: synthesise + guardrail pass
  SCANNING --> DEGRADED: a worker timed out
  SCANNING --> BLOCKED: guardrail fail
  DEGRADED --> [*]
  BLOCKED --> [*]
  SYNTHESISED --> SYNTHESISED: DigestEvalScored
  SYNTHESISED --> [*]
```

## Entity model

```mermaid
erDiagram
  RESEARCH_DIGEST ||--o{ PUBLICATION_BUNDLE : has
  RESEARCH_DIGEST ||--o{ TREND_REPORT : has
  RESEARCH_DIGEST ||--o| SYNTHESISED_DIGEST : produces
  QUERY_QUEUE ||--|| RESEARCH_DIGEST : seeds
  RESEARCH_DIGEST {
    string digestId
    string query
    enum status
    int evalScore
    instant createdAt
  }
  QUERY_QUEUE {
    string digestId
    string query
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `LiteratureCoordinator` | AutonomousAgent | `application/LiteratureCoordinator.java` |
| `PaperScout` | AutonomousAgent | `application/PaperScout.java` |
| `DomainAnalyst` | AutonomousAgent | `application/DomainAnalyst.java` |
| `AcademicResearchTasks` | Task constants | `application/AcademicResearchTasks.java` |
| `ResearchWorkflow` | Workflow | `application/ResearchWorkflow.java` |
| `ResearchDigestEntity` | EventSourcedEntity | `domain/ResearchDigestEntity.java` |
| `QueryQueue` | EventSourcedEntity | `domain/QueryQueue.java` |
| `DigestView` | View | `application/DigestView.java` |
| `QueryRequestConsumer` | Consumer | `application/QueryRequestConsumer.java` |
| `QuerySimulator` | TimedAction | `application/QuerySimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `DigestEndpoint` | HttpEndpoint | `api/DigestEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `scoutStep` and `analyseStep` get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `scoutStep` and `analyseStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Idempotency:** the workflow id is the `digestId`. Re-delivery of the same `QuerySubmitted` event resolves to the same workflow instance — no duplicate digest.
- **Degrade path (compensation):** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output exists and ends with `DigestDegraded`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `DigestView.getAllDigests` (no enum WHERE clause — Lesson 2) and filters client-side for the oldest `SYNTHESISED` digest lacking an `evalScore`.
