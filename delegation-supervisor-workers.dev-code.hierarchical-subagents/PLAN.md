# PLAN — Hierarchical Sub-Agent Delegation

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  TE[TaskEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  TQ[TaskQueue<br/>EventSourcedEntity]:::ese
  TC[TaskQueueConsumer<br/>Consumer]:::con
  WF[DelegationWorkflow<br/>Workflow]:::wf
  TO[TaskOrchestrator<br/>AutonomousAgent]:::ag
  AS[ArchitectSpecialist<br/>AutonomousAgent]:::ag
  IS[ImplementationSpecialist<br/>AutonomousAgent]:::ag
  DE[TaskDelegationEntity<br/>EventSourcedEntity]:::ese
  VW[DelegationView<br/>View]:::vw
  SIM[TaskSimulator<br/>TimedAction]:::ta
  CE[ContributionEvaluator<br/>TimedAction]:::ta

  TE -->|POST /tasks| TQ
  SIM -.->|every 60s| TQ
  TQ -.->|TaskSubmitted| TC
  TC -->|start workflow| WF
  WF -->|DECOMPOSE| TO
  WF -->|validate input schema| WF
  WF -->|DESIGN| AS
  WF -->|OUTLINE| IS
  WF -->|SYNTHESISE| TO
  WF -->|commands| DE
  DE -.->|events| VW
  CE -.->|every 5m| VW
  CE -->|recordContributionScores| DE
  TE -->|getAllTasks / SSE| VW
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
  participant WF as DelegationWorkflow
  participant TO as TaskOrchestrator
  participant AS as ArchitectSpecialist
  participant IS as ImplementationSpecialist
  participant DE as TaskDelegationEntity

  U->>TE: POST /api/tasks {description}
  TE->>TQ: enqueueTask
  TQ-->>WF: TaskQueueConsumer starts workflow
  WF->>DE: createTask (SCOPING)
  WF->>TO: DECOMPOSE -> DelegationPlan
  WF->>WF: guardrailStep validates DelegationPlan schema
  alt schema invalid
    WF->>DE: reject (REJECTED)
  else schema valid
    WF->>DE: status DELEGATING
    par parallel fan-out
      WF->>AS: DESIGN -> DesignProposal
    and
      WF->>IS: OUTLINE -> ImplementationOutline
    end
    Note over WF: join; if either step times out (60s) -> degradeStep
    WF->>WF: evaluateStep scores contributions (1-5)
    WF->>TO: SYNTHESISE(design, implementation) -> SynthesisedSolution
    WF->>WF: validateStep vets the solution
    alt validate passes
      WF->>DE: synthesise (SYNTHESISED)
    else validate fails
      WF->>DE: reject (REJECTED)
    end
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> SCOPING
  SCOPING --> REJECTED: schema validation fail
  SCOPING --> DELEGATING: DelegationPlan valid
  DELEGATING --> SYNTHESISED: synthesise + validate pass
  DELEGATING --> DEGRADED: a specialist timed out
  DELEGATING --> REJECTED: validate fail
  DEGRADED --> [*]
  REJECTED --> [*]
  SYNTHESISED --> SYNTHESISED: ContributionScored
  SYNTHESISED --> [*]
```

## Entity model

```mermaid
erDiagram
  DELEGATION_TASK ||--o{ DESIGN_PROPOSAL : has
  DELEGATION_TASK ||--o{ IMPLEMENTATION_OUTLINE : has
  DELEGATION_TASK ||--o| SYNTHESISED_SOLUTION : produces
  TASK_QUEUE ||--|| DELEGATION_TASK : seeds
  DELEGATION_TASK {
    string taskId
    string description
    enum status
    int architectScore
    int implementationScore
    instant createdAt
  }
  TASK_QUEUE {
    string taskId
    string description
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `TaskOrchestrator` | AutonomousAgent | `application/TaskOrchestrator.java` |
| `ArchitectSpecialist` | AutonomousAgent | `application/ArchitectSpecialist.java` |
| `ImplementationSpecialist` | AutonomousAgent | `application/ImplementationSpecialist.java` |
| `DelegationTasks` | Task constants | `application/DelegationTasks.java` |
| `DelegationWorkflow` | Workflow | `application/DelegationWorkflow.java` |
| `TaskDelegationEntity` | EventSourcedEntity | `domain/TaskDelegationEntity.java` |
| `TaskQueue` | EventSourcedEntity | `domain/TaskQueue.java` |
| `DelegationView` | View | `application/DelegationView.java` |
| `TaskQueueConsumer` | Consumer | `application/TaskQueueConsumer.java` |
| `TaskSimulator` | TimedAction | `application/TaskSimulator.java` |
| `ContributionEvaluator` | TimedAction | `application/ContributionEvaluator.java` |
| `TaskEndpoint` | HttpEndpoint | `api/TaskEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `designStep` and `outlineStep` get 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `designStep` and `outlineStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Input guardrail placement:** the `guardrailStep` runs after `decomposeStep` and before the parallel fan-out, so a malformed `DelegationPlan` stops the workflow before any specialist is invoked.
- **Idempotency:** the workflow id is the `taskId`. Re-delivery of the same `TaskSubmitted` event resolves to the same workflow instance — no duplicate task.
- **Degrade path (compensation):** if either specialist times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from whichever partial output exists and ends with `TaskDegraded`. No infinite retry.
- **Contribution scoring:** `ContributionEvaluator` reads `DelegationView.getAllTasks` (no enum WHERE clause — Lesson 2) and filters client-side for `SYNTHESISED` tasks without `architectScore`.
