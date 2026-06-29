# PLAN — Supervisor with Subagents

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  TE[TaskEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  TQ[TaskQueue<br/>EventSourcedEntity]:::ese
  TC[TaskRequestConsumer<br/>Consumer]:::con
  WF[TaskOrchestrationWorkflow<br/>Workflow]:::wf
  SV[TaskSupervisor<br/>AutonomousAgent]:::ag
  DA[DataSubagent<br/>AutonomousAgent]:::ag
  SA[SummarySubagent<br/>AutonomousAgent]:::ag
  TX[TaskEntity<br/>EventSourcedEntity]:::ese
  TV[TaskView<br/>View]:::vw
  SIM[TaskSimulator<br/>TimedAction]:::ta
  EV[RoutingEvalSampler<br/>TimedAction]:::ta

  TE -->|POST /tasks| TQ
  SIM -.->|every 60s| TQ
  TQ -.->|TaskSubmitted| TC
  TC -->|start workflow| WF
  WF -->|ROUTE| SV
  WF -->|guardrail check| WF
  WF -->|FETCH_DATA| DA
  WF -->|SUMMARISE| SA
  WF -->|ASSEMBLE| SV
  WF -->|commands| TX
  TX -.->|events| TV
  EV -.->|every 5m| TV
  EV -->|recordEval| TX
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

Solid arrows: synchronous commands. Dashed arrows: event subscriptions. Dotted arrows: scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant TE as TaskEndpoint
  participant TQ as TaskQueue
  participant WF as TaskOrchestrationWorkflow
  participant SV as TaskSupervisor
  participant DA as DataSubagent
  participant SA as SummarySubagent
  participant TX as TaskEntity

  U->>TE: POST /api/tasks {description}
  TE->>TQ: submitTask
  TQ-->>WF: TaskRequestConsumer starts workflow
  WF->>TX: createTask (RECEIVED)
  WF->>SV: ROUTE -> RoutingPlan
  WF->>TX: status ROUTING
  WF->>WF: guardrailStep checks RoutingPlan
  alt guardrail fails
    WF->>TX: block (BLOCKED)
  else guardrail passes
    WF->>TX: status IN_PROGRESS
    par parallel fan-out
      WF->>DA: FETCH_DATA -> DataBundle
    and
      WF->>SA: SUMMARISE -> SummaryOutput
    end
    Note over WF: join; if either step times out (60s) -> degradeStep
    WF->>SV: ASSEMBLE(data, summary) -> TaskResult
    WF->>TX: completeTask (COMPLETED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> ROUTING: task created, routing started
  ROUTING --> BLOCKED: guardrail fails
  ROUTING --> IN_PROGRESS: guardrail passes
  IN_PROGRESS --> COMPLETED: assembly succeeds
  IN_PROGRESS --> DEGRADED: a subagent timed out
  DEGRADED --> [*]
  BLOCKED --> [*]
  COMPLETED --> COMPLETED: RoutingEvalScored
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  TASK_ENTITY ||--o| ROUTING_PLAN : has
  TASK_ENTITY ||--o| DATA_BUNDLE : has
  TASK_ENTITY ||--o| SUMMARY_OUTPUT : has
  TASK_ENTITY ||--o| TASK_RESULT : produces
  TASK_QUEUE ||--|| TASK_ENTITY : seeds
  TASK_ENTITY {
    string taskId
    string description
    enum status
    int evalScore
    instant createdAt
  }
  TASK_QUEUE {
    string taskId
    string description
    string submittedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `TaskSupervisor` | AutonomousAgent | `application/TaskSupervisor.java` |
| `DataSubagent` | AutonomousAgent | `application/DataSubagent.java` |
| `SummarySubagent` | AutonomousAgent | `application/SummarySubagent.java` |
| `SupervisorTasks` | Task constants | `application/SupervisorTasks.java` |
| `TaskOrchestrationWorkflow` | Workflow | `application/TaskOrchestrationWorkflow.java` |
| `TaskEntity` | EventSourcedEntity | `domain/TaskEntity.java` |
| `TaskQueue` | EventSourcedEntity | `domain/TaskQueue.java` |
| `TaskView` | View | `application/TaskView.java` |
| `TaskRequestConsumer` | Consumer | `application/TaskRequestConsumer.java` |
| `TaskSimulator` | TimedAction | `application/TaskSimulator.java` |
| `RoutingEvalSampler` | TimedAction | `application/RoutingEvalSampler.java` |
| `TaskEndpoint` | HttpEndpoint | `api/TaskEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `fetchDataStep` and `summariseStep` get 60s; `assembleStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `fetchDataStep` and `summariseStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Guardrail placement:** `guardrailStep` runs after `routeStep` and before the parallel fan-out. If the routing plan is rejected, neither subagent is invoked — the task is blocked immediately, avoiding wasted compute.
- **Idempotency:** the workflow id is the `taskId`. Re-delivery of the same `TaskSubmitted` event resolves to the same workflow instance — no duplicate task.
- **Degrade path (compensation):** if either subagent times out, `defaultStepRecovery` routes to `degradeStep`, which assembles from whichever partial output exists and ends with `TaskDegraded`. No infinite retry.
- **Eval sampling:** `RoutingEvalSampler` reads `TaskView.getAllTasks` (no enum WHERE clause — Lesson 2) and filters client-side for the oldest `COMPLETED` task lacking an `evalScore`.
