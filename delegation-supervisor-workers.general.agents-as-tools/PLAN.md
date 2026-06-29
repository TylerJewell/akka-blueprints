# PLAN — Agents as Tools

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  TE[TaskEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  TQ[TaskQueue<br/>EventSourcedEntity]:::ese
  RC[TaskRequestConsumer<br/>Consumer]:::con
  WF[TaskWorkflow<br/>Workflow]:::wf
  SV[TaskSupervisor<br/>AutonomousAgent]:::ag
  WR[WriterAgent<br/>AutonomousAgent]:::ag
  DA[DataAgent<br/>AutonomousAgent]:::ag
  TR[TaskRecordEntity<br/>EventSourcedEntity]:::ese
  TV[TaskView<br/>View]:::vw
  SIM[TaskSimulator<br/>TimedAction]:::ta
  QS[QualitySampler<br/>TimedAction]:::ta

  TE -->|POST /tasks| TQ
  SIM -.->|every 60s| TQ
  TQ -.->|TaskSubmitted| RC
  RC -->|start workflow| WF
  WF -->|SUPERVISE| SV
  SV -->|writer_tool| WR
  SV -->|data_tool| DA
  WF -->|commands| TR
  TR -.->|events| TV
  QS -.->|every 5m| TV
  QS -->|recordQuality| TR
  TE -->|getAllTasks / SSE| TV
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

Solid arrows: synchronous commands or tool calls. Dashed arrows: event subscriptions. Dotted arrows: scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant TE as TaskEndpoint
  participant TQ as TaskQueue
  participant WF as TaskWorkflow
  participant SV as TaskSupervisor
  participant WR as WriterAgent
  participant DA as DataAgent
  participant TR as TaskRecordEntity

  U->>TE: POST /api/tasks {description}
  TE->>TQ: submitTask
  TQ-->>WF: TaskRequestConsumer starts workflow
  WF->>TR: createTask (QUEUED)
  WF->>TR: markProcessing (PROCESSING)
  WF->>SV: SUPERVISE -> AssembledResult
  note over SV: selects tools based on task description
  SV->>WR: writer_tool -> DraftOutput
  SV->>DA: data_tool -> DataOutput
  SV-->>WF: AssembledResult{answer, toolsUsed}
  alt result assembled
    WF->>TR: completeTask (COMPLETED)
  else supervisor rejected task
    WF->>TR: rejectTask (REJECTED)
  else supervise step timed out
    WF->>TR: failTask (FAILED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> PROCESSING: workflow starts
  PROCESSING --> COMPLETED: supervisor assembles result
  PROCESSING --> REJECTED: supervisor cannot route task
  PROCESSING --> FAILED: supervise step timed out
  REJECTED --> [*]
  FAILED --> [*]
  COMPLETED --> COMPLETED: QualityScored
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  TASK_RECORD ||--o| DRAFT_OUTPUT : has
  TASK_RECORD ||--o| DATA_OUTPUT : has
  TASK_RECORD ||--o| ASSEMBLED_RESULT : produces
  TASK_QUEUE ||--|| TASK_RECORD : seeds
  TASK_RECORD {
    string taskId
    string description
    enum status
    int qualityScore
    instant createdAt
  }
  TASK_QUEUE {
    string taskId
    string description
    string requestedBy
    instant submittedAt
  }
  DATA_OUTPUT {
    list entities
    instant summarisedAt
  }
  ASSEMBLED_RESULT {
    string answer
    list toolsUsed
    instant assembledAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `TaskSupervisor` | AutonomousAgent | `application/TaskSupervisor.java` |
| `WriterAgent` | AutonomousAgent | `application/WriterAgent.java` |
| `DataAgent` | AutonomousAgent | `application/DataAgent.java` |
| `AgentTasks` | Task constants | `application/AgentTasks.java` |
| `TaskWorkflow` | Workflow | `application/TaskWorkflow.java` |
| `TaskRecordEntity` | EventSourcedEntity | `domain/TaskRecordEntity.java` |
| `TaskQueue` | EventSourcedEntity | `domain/TaskQueue.java` |
| `TaskView` | View | `application/TaskView.java` |
| `TaskRequestConsumer` | Consumer | `application/TaskRequestConsumer.java` |
| `TaskSimulator` | TimedAction | `application/TaskSimulator.java` |
| `QualitySampler` | TimedAction | `application/QualitySampler.java` |
| `TaskEndpoint` | HttpEndpoint | `api/TaskEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `superviseStep` gets 120s to allow the supervisor to call both tools in sequence. Each underlying tool call is bounded by the supervisor's iteration budget (45 s per tool). The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Tool call model:** unlike parallel fan-out blueprints, the supervisor calls tools sequentially inside its own iteration loop. The workflow sees one `SUPERVISE` step returning the assembled result; it does not fork separate steps per tool.
- **Idempotency:** the workflow id is the `taskId`. Re-delivery of the same `TaskSubmitted` event resolves to the same workflow instance — no duplicate task record.
- **Rejection path:** the supervisor returns a rejection indicator in `AssembledResult` (e.g., `toolsUsed` is empty and `answer` starts with "REJECTED:"). The workflow inspects this and routes to `rejectStep` rather than `persistStep`.
- **Quality sampling:** `QualitySampler` reads `TaskView.getAllTasks` (no enum WHERE clause) and filters client-side for the oldest `COMPLETED` task lacking a `qualityScore`.
