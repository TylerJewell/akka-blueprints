# PLAN — Concurrent Research Writer

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  RE[ReportEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  TQ[TopicQueue<br/>EventSourcedEntity]:::ese
  RC[ReportRequestConsumer<br/>Consumer]:::con
  WF[ReportWorkflow<br/>Workflow]:::wf
  CO[ReportCoordinator<br/>AutonomousAgent]:::ag
  SR[SourceResearcher<br/>AutonomousAgent]:::ag
  DW[DraftWriter<br/>AutonomousAgent]:::ag
  RE2[ReportEntity<br/>EventSourcedEntity]:::ese
  VW[ReportView<br/>View]:::vw
  SIM[TopicSimulator<br/>TimedAction]:::ta
  QS[QualitySampler<br/>TimedAction]:::ta

  RE -->|POST /reports| TQ
  SIM -.->|every 60s| TQ
  TQ -.->|TopicSubmitted| RC
  RC -->|start workflow| WF
  WF -->|PLAN_REPORT| CO
  WF -->|GATHER_SOURCES| SR
  WF -->|WRITE_DRAFT| DW
  WF -->|ASSEMBLE_REPORT| CO
  WF -->|commands| RE2
  RE2 -.->|events| VW
  QS -.->|every 5m| VW
  QS -->|recordScore| RE2
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
  participant TQ as TopicQueue
  participant WF as ReportWorkflow
  participant CO as ReportCoordinator
  participant SR as SourceResearcher
  participant DW as DraftWriter
  participant RE2 as ReportEntity

  U->>RE: POST /api/reports {topic}
  RE->>TQ: enqueueTopic
  TQ-->>WF: ReportRequestConsumer starts workflow
  WF->>RE2: createReport (PLANNING)
  WF->>CO: PLAN_REPORT -> WritingPlan
  WF->>RE2: status IN_PROGRESS
  par parallel fan-out
    WF->>SR: GATHER_SOURCES -> SourceBundle
  and
    WF->>DW: WRITE_DRAFT -> DraftSection
  end
  Note over WF: join; if either step times out (60s) -> degradeStep
  WF->>CO: ASSEMBLE_REPORT(sources, draft) -> AssembledReport
  WF->>WF: guardrailStep vets the report
  alt guardrail passes
    WF->>RE2: assemble (ASSEMBLED)
  else guardrail fails
    WF->>RE2: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> IN_PROGRESS: WritingPlan ready
  IN_PROGRESS --> ASSEMBLED: assemble + guardrail pass
  IN_PROGRESS --> DEGRADED: a worker timed out
  IN_PROGRESS --> BLOCKED: guardrail fail
  DEGRADED --> [*]
  BLOCKED --> [*]
  ASSEMBLED --> ASSEMBLED: ReportScored
  ASSEMBLED --> [*]
```

## Entity model

```mermaid
erDiagram
  REPORT ||--o{ SOURCE_BUNDLE : has
  REPORT ||--o{ DRAFT_SECTION : has
  REPORT ||--o| ASSEMBLED_REPORT : produces
  TOPIC_QUEUE ||--|| REPORT : seeds
  REPORT {
    string reportId
    string topic
    enum status
    int qualityScore
    instant createdAt
  }
  TOPIC_QUEUE {
    string reportId
    string topic
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `ReportCoordinator` | AutonomousAgent | `application/ReportCoordinator.java` |
| `SourceResearcher` | AutonomousAgent | `application/SourceResearcher.java` |
| `DraftWriter` | AutonomousAgent | `application/DraftWriter.java` |
| `ReportTasks` | Task constants | `application/ReportTasks.java` |
| `ReportWorkflow` | Workflow | `application/ReportWorkflow.java` |
| `ReportEntity` | EventSourcedEntity | `domain/ReportEntity.java` |
| `TopicQueue` | EventSourcedEntity | `domain/TopicQueue.java` |
| `ReportView` | View | `application/ReportView.java` |
| `ReportRequestConsumer` | Consumer | `application/ReportRequestConsumer.java` |
| `TopicSimulator` | TimedAction | `application/TopicSimulator.java` |
| `QualitySampler` | TimedAction | `application/QualitySampler.java` |
| `ReportEndpoint` | HttpEndpoint | `api/ReportEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `researchStep` and `draftStep` each get 60s; `assembleStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `researchStep` and `draftStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Idempotency:** the workflow id is the `reportId`. Re-delivery of the same `TopicSubmitted` event resolves to the same workflow instance — no duplicate report.
- **Degrade path (compensation):** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which assembles from whichever partial output exists and ends with `ReportDegraded`. No infinite retry.
- **Quality sampling:** `QualitySampler` reads `ReportView.getAllReports` (no enum WHERE clause — Lesson 2) and filters client-side for the oldest `ASSEMBLED` report lacking a `qualityScore`.
