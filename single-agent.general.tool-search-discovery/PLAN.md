# PLAN — tool-search-discovery

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef reg fill:#0a1a1a,stroke:#22d3ee,color:#22d3ee;

  API[TaskEndpoint]:::ep
  Entity[TaskEntity]:::ese
  Catalog[ToolCatalogConsumer]:::cons
  Registry[ToolRegistry]:::reg
  WF[DiscoveryWorkflow]:::wf
  Agent[DiscoveryAgent]:::agent
  Guard[ToolAllowlistGuardrail]:::guard
  View[TaskView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  Entity -.->|TaskReceived| View
  Catalog -->|merge entries| Registry
  WF -->|discoverStep search| Registry
  WF -->|recordDiscovered| Entity
  WF -->|executeStep runSingleTask| Agent
  Agent -.->|before-tool-invocation| Guard
  Agent -->|TaskResult| WF
  WF -->|completeTask| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as TaskEndpoint
  participant E as TaskEntity
  participant W as DiscoveryWorkflow
  participant R as ToolRegistry
  participant A as DiscoveryAgent
  participant G as ToolAllowlistGuardrail

  U->>API: POST /api/tasks
  API->>E: submit(request)
  E-->>API: { taskId }
  API->>W: start(taskId)
  W->>R: search(userQuery)
  R-->>W: List<ToolEntry>
  W->>E: recordDiscovered(tools)
  W->>A: runSingleTask(query + catalog attachment)
  A->>G: before-tool-invocation(candidateTool)
  G-->>A: accept (or strip + reject)
  A-->>W: TaskResult
  W->>E: completeTask(result)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> DISCOVERING: TaskReceived (workflow starts)
  DISCOVERING --> READY: ToolsDiscovered
  READY --> EXECUTING: ExecutionStarted
  EXECUTING --> COMPLETED: TaskCompleted
  EXECUTING --> FAILED: TaskFailed (agent error)
  DISCOVERING --> FAILED: TaskFailed (registry error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ TaskReceived : emits
  TaskEntity ||--o{ ToolsDiscovered : emits
  TaskEntity ||--o{ ExecutionStarted : emits
  TaskEntity ||--o{ TaskCompleted : emits
  TaskEntity ||--o{ TaskFailed : emits
  TaskView }o--|| TaskEntity : projects
  ToolCatalogConsumer }o--|| ToolRegistry : updates
  DiscoveryWorkflow }o--|| TaskEntity : reads-and-writes
  DiscoveryWorkflow }o--|| ToolRegistry : queries
  DiscoveryAgent ||--o{ TaskResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TaskEndpoint` | `api/TaskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/TaskRecord.java`, events in `domain/TaskEvent.java`) |
| `ToolCatalogConsumer` | `application/ToolCatalogConsumer.java` |
| `DiscoveryWorkflow` | `application/DiscoveryWorkflow.java` |
| `DiscoveryAgent` | `application/DiscoveryAgent.java` (tasks in `application/DiscoveryTasks.java`) |
| `ToolAllowlistGuardrail` | `application/ToolAllowlistGuardrail.java` |
| `ToolRegistry` | `application/ToolRegistry.java` |
| `TaskView` | `application/TaskView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `discoverStep` 15 s, `executeStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(DiscoveryWorkflow::error)`. The 60 s on `executeStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"discovery-" + taskId` as the workflow id. `TaskEntity.recordDiscovered` is event-version-guarded — a redelivered `discoverStep` is a no-op against an already-READY entity.
- **One agent per task**: the AutonomousAgent instance id is `"discovery-" + taskId`, giving each task its own conversation context.
- **Allowlist loaded once per guardrail instance**: `allowlist.json` is read from classpath at service start. Changes require a restart (the file is not hot-reloaded in the base sample; a deployer may extend `ToolCatalogConsumer` to republish the allowlist).
- **ToolRegistry is in-process**: the registry is a `ConcurrentHashMap` seeded at startup and updated by `ToolCatalogConsumer`. It is not distributed — each node has its own copy. For multi-node production deployment a deployer would replace it with an Akka Key-Value Entity.
- **No saga / no compensation**: `discoverStep` is a pure registry read; `executeStep` appends events. There is nothing external to roll back.
