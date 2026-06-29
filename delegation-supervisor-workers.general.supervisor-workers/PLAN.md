# PLAN — Multi-Agent Collaboration (Supervisor)

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  CE[CollaborationEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  TQ[TaskRequestQueue<br/>EventSourcedEntity]:::ese
  TC[TaskRequestConsumer<br/>Consumer]:::con
  WF[CollaborationWorkflow<br/>Workflow]:::wf
  SV[TaskSupervisor<br/>AutonomousAgent]:::ag
  RW[ResearchWorker<br/>AutonomousAgent]:::ag
  CW[ChartWorker<br/>AutonomousAgent]:::ag
  TE[TaskEntity<br/>EventSourcedEntity]:::ese
  TV[TaskView<br/>View]:::vw
  SIM[TaskSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  CE -->|POST /tasks| TQ
  SIM -.->|every 60s| TQ
  TQ -.->|TaskSubmitted| TC
  TC -->|start workflow| WF
  WF -->|ROUTE| SV
  WF -->|RESEARCH| RW
  WF -->|CHART| CW
  WF -->|SYNTHESISE| SV
  WF -->|commands| TE
  TE -.->|events| TV
  EV -.->|every 5m| TV
  EV -->|recordEval| TE
  CE -->|getAllTasks / SSE| TV
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
  participant CE as CollaborationEndpoint
  participant TQ as TaskRequestQueue
  participant WF as CollaborationWorkflow
  participant SV as TaskSupervisor
  participant RW as ResearchWorker
  participant CW as ChartWorker
  participant TE as TaskEntity

  U->>CE: POST /api/tasks {description}
  CE->>TQ: enqueueTask
  TQ-->>WF: TaskRequestConsumer starts workflow
  WF->>TE: createTask (ROUTING)
  WF->>SV: ROUTE -> RoutingDecision
  WF->>TE: routeTask (IN_PROGRESS)
  par parallel fan-out
    WF->>RW: RESEARCH -> ResearchOutput
  and
    WF->>CW: CHART -> ChartData
  end
  Note over WF: join; if either step times out (60s) -> degradeStep
  WF->>SV: SYNTHESISE(research, chart) -> TaskResult
  WF->>WF: guardrailStep checks tool calls
  alt guardrail passes
    WF->>TE: completeTask (COMPLETED)
  else guardrail fails
    WF->>TE: blockTask (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> ROUTING
  ROUTING --> IN_PROGRESS: RoutingDecision ready
  IN_PROGRESS --> COMPLETED: synthesis + guardrail pass
  IN_PROGRESS --> DEGRADED: a worker timed out
  IN_PROGRESS --> BLOCKED: guardrail fail
  DEGRADED --> [*]
  BLOCKED --> [*]
  COMPLETED --> COMPLETED: TaskEvalScored
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  COLLABORATION_TASK ||--o| ROUTING_DECISION : has
  COLLABORATION_TASK ||--o| RESEARCH_OUTPUT : has
  COLLABORATION_TASK ||--o| CHART_DATA : has
  COLLABORATION_TASK ||--o| TASK_RESULT : produces
  TASK_REQUEST_QUEUE ||--|| COLLABORATION_TASK : seeds
  COLLABORATION_TASK {
    string taskId
    string description
    enum status
    int evalScore
    instant createdAt
  }
  TASK_REQUEST_QUEUE {
    string taskId
    string description
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `TaskSupervisor` | AutonomousAgent | `application/TaskSupervisor.java` |
| `ResearchWorker` | AutonomousAgent | `application/ResearchWorker.java` |
| `ChartWorker` | AutonomousAgent | `application/ChartWorker.java` |
| `WorkerTasks` | Task constants | `application/WorkerTasks.java` |
| `CollaborationWorkflow` | Workflow | `application/CollaborationWorkflow.java` |
| `TaskEntity` | EventSourcedEntity | `domain/TaskEntity.java` |
| `TaskRequestQueue` | EventSourcedEntity | `domain/TaskRequestQueue.java` |
| `TaskView` | View | `application/TaskView.java` |
| `TaskRequestConsumer` | Consumer | `application/TaskRequestConsumer.java` |
| `TaskSimulator` | TimedAction | `application/TaskSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `CollaborationEndpoint` | HttpEndpoint | `api/CollaborationEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `researchStep` and `chartStep` get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `researchStep` and `chartStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Idempotency:** the workflow id is the `taskId`. Re-delivery of the same `TaskSubmitted` event resolves to the same workflow instance — no duplicate task.
- **Degrade path (compensation):** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output exists and ends with `TaskDegraded`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `TaskView.getAllTasks` (no enum WHERE clause — Lesson 2) and filters client-side for the oldest `COMPLETED` task lacking an `evalScore`.
- **Before-tool-call guardrail:** runs on `TaskSupervisor` before any tool invocation. The permitted list is `[SEARCH, CHART_GENERATE]`; any other tool call causes an immediate block.
