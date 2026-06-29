# PLAN — multi-agent-handoff-workflow

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'fontFamily':'Instrument Sans, system-ui, sans-serif'
}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef autonomous fill:#0e2a1e,stroke:#3fb950,color:#3fb950;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Sim[TaskSimulator]:::ta
  Queue[TaskQueue]:::ese
  Admit[AdmissionGuardrail]:::cons
  Router[RouterAgent]:::agent
  Analyst[DataAnalyst]:::autonomous
  Writer[ContentWriter]:::autonomous
  Reviewer[CodeReviewer]:::autonomous
  Judge[RoutingJudge]:::agent
  WF[HandoffWorkflow]:::wf
  Entity[TaskEntity]:::ese
  View[TaskView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[WorkflowEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Admit
  Admit -->|register + admit| Entity
  Admit -->|start workflow| WF
  WF -->|route| Router
  WF -->|EXECUTE task| Analyst
  WF -->|EXECUTE task| Writer
  WF -->|EXECUTE task| Reviewer
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordRoutingScore| Entity
  API -->|submit / query / SSE| Entity
  API -->|query| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (data-analysis happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as TaskSimulator
  participant Q as TaskQueue
  participant A as AdmissionGuardrail
  participant E as TaskEntity
  participant W as HandoffWorkflow
  participant R as RouterAgent
  participant D as DataAnalyst
  participant Sc as RoutingEvalScorer
  participant J as RoutingJudge

  Sim->>Q: receive(IncomingTask)
  Q->>A: InboundTaskReceived
  A->>E: registerIncoming + recordAdmission(admitted=true)
  A->>W: start(taskId, incoming)
  W->>R: route(incoming)
  R-->>W: RoutingDecision{DATA_ANALYSIS}
  W->>E: recordRouting(decision) [emits RoutingDecided]
  E->>Sc: RoutingDecided event
  Sc->>J: score(incoming, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>E: recordTaskRouted(DATA_ANALYSIS) [emits TaskRouted]
  W->>D: runSingleTask(EXECUTE, prompt)
  D-->>W: TaskResult
  W->>E: recordDraft(result) [emits ResultDrafted]
  W->>W: validateStep (non-empty headline, body length, specialistTag)
  W->>E: publish(result) [emits ResultPublished, status COMPLETED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream, independent of `HandoffWorkflow`. Both writes target the same `TaskEntity`; the entity's commands are idempotent on `taskId`.

## State machine — `TaskEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> ADMITTED: TaskAdmitted
  RECEIVED --> REJECTED: TaskRejectedAtAdmission
  ADMITTED --> ROUTED: RoutingDecided
  ROUTED --> ROUTED_DATA_ANALYSIS: domain = DATA_ANALYSIS
  ROUTED --> ROUTED_CONTENT_WRITING: domain = CONTENT_WRITING
  ROUTED --> ROUTED_CODE_REVIEW: domain = CODE_REVIEW
  ROUTED --> REJECTED: domain = UNROUTABLE
  ROUTED_DATA_ANALYSIS --> EXECUTING: specialist started
  ROUTED_CONTENT_WRITING --> EXECUTING: specialist started
  ROUTED_CODE_REVIEW --> EXECUTING: specialist started
  EXECUTING --> TOOL_BLOCKED: ToolCallBlocked
  EXECUTING --> VALIDATION_FAILED: ValidationFailed
  EXECUTING --> COMPLETED: ResultPublished
  TOOL_BLOCKED --> [*]
  VALIDATION_FAILED --> [*]
  COMPLETED --> [*]
  REJECTED --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result. The state machine therefore treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  TaskEntity ||--o{ TaskReceived : emits
  TaskEntity ||--o{ TaskAdmitted : emits
  TaskEntity ||--o{ TaskRejectedAtAdmission : emits
  TaskEntity ||--o{ RoutingDecided : emits
  TaskEntity ||--o{ TaskRouted : emits
  TaskEntity ||--o{ ToolCallBlocked : emits
  TaskEntity ||--o{ ResultDrafted : emits
  TaskEntity ||--o{ ValidationFailed : emits
  TaskEntity ||--o{ ResultPublished : emits
  TaskEntity ||--o{ TaskRejected : emits
  TaskEntity ||--o{ RoutingScored : emits
  TaskView }o--|| TaskEntity : projects
  TaskQueue ||--o{ InboundTaskReceived : emits
  AdmissionGuardrail }o--|| TaskQueue : subscribes
  RoutingEvalScorer }o--|| TaskEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TaskSimulator` | `application/TaskSimulator.java` |
| `TaskQueue` | `application/TaskQueue.java` |
| `AdmissionGuardrail` | `application/AdmissionGuardrail.java` |
| `RouterAgent` | `application/RouterAgent.java` |
| `DataAnalyst` | `application/DataAnalyst.java` |
| `ContentWriter` | `application/ContentWriter.java` |
| `CodeReviewer` | `application/CodeReviewer.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `HandoffWorkflow` | `application/HandoffWorkflow.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `TaskView` | `application/TaskView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `WorkflowEndpoint` | `api/WorkflowEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/WorkflowTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `routeStep` 20 s, `validateStep` 20 s, `dataStep` / `contentStep` / `codeStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the task to `REJECTED` with the failure reason captured.
- **Idempotency.** Every per-task primitive is keyed by `taskId`: `TaskEntity` id is `taskId`; `HandoffWorkflow` id is `taskId`; agent sessions for `RouterAgent`, `RoutingJudge` use `taskId`. Duplicate admission events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `RoutingEvalScorer` (Consumer) and `HandoffWorkflow` both append events to the same `TaskEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `TaskResult`, the workflow either publishes or records a validation failure. There is no rollback path.
- **No HITL on the happy path.** The system only surfaces blocked tool calls or validation failures for operator review; everything else flows through to `COMPLETED` autonomously.
- **Simulator throughput.** `TaskSimulator` drips one request every 30 s; the system can comfortably process each task end-to-end inside that window with mock or real LLMs.
