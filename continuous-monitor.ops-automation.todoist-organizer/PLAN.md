# PLAN — todoist-organizer

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Poller[TodoistPoller]:::ta
  Queue[TaskQueue]:::ese
  Consumer[TaskFetchConsumer]:::cons
  Classifier[TaskClassifierAgent]:::agent
  WF[OrganizerWorkflow]:::wf
  Entity[TodoistTaskEntity]:::ese
  View[OrganizerView]:::view
  EvalRunner[EvalRunner]:::ta
  API[OrganizerEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 30s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|registerTask| Entity
  Consumer -->|start| WF
  WF -->|call| Classifier
  WF -->|guardrail check| WF
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|query/SSE| View
  EvalRunner -.->|every 60m| Entity
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as TodoistPoller
  participant Q as TaskQueue
  participant C as TaskFetchConsumer
  participant E as TodoistTaskEntity
  participant W as OrganizerWorkflow
  participant CL as TaskClassifierAgent
  participant U as User (UI)
  participant API as OrganizerEndpoint

  P->>Q: emit TaskFetched
  Q->>C: TaskFetched
  C->>E: registerTask
  C->>W: start({taskId, task})
  W->>CL: classify(task)
  CL-->>W: ClassificationResult(confidence=high)
  W->>E: emit TaskClassified
  W->>W: guardrailCheck(confidence, projectAllowList)
  W->>E: emit GuardrailChecked(allowed=true)
  W->>E: emit TaskUpdated
  Note over E,U: Task in UPDATED state
  U->>API: GET /api/organizer/{id}
  API-->>U: OrganizerTask (status UPDATED, classification visible)
```

## State machine — `TodoistTaskEntity`

```mermaid
stateDiagram-v2
  [*] --> FETCHED
  FETCHED --> CLASSIFIED: TaskClassified
  CLASSIFIED --> GUARDRAIL_BLOCKED: GuardrailChecked(allowed=false)
  CLASSIFIED --> UPDATED: GuardrailChecked(allowed=true) + TaskUpdated
  CLASSIFIED --> SKIPPED: classification=low, explicit skip
  UPDATED --> UPDATED: EvalScored
  GUARDRAIL_BLOCKED --> [*]
  SKIPPED --> [*]
  UPDATED --> [*]
  CLASSIFIED --> FAILED: workflow error
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TodoistTaskEntity ||--o{ TaskFetched : emits
  TodoistTaskEntity ||--o{ TaskClassified : emits
  TodoistTaskEntity ||--o{ GuardrailChecked : emits
  TodoistTaskEntity ||--o{ TaskUpdated : emits
  TodoistTaskEntity ||--o{ TaskSkipped : emits
  TodoistTaskEntity ||--o{ TaskFailed : emits
  TodoistTaskEntity ||--o{ EvalScored : emits
  OrganizerView }o--|| TodoistTaskEntity : projects
  TaskQueue ||--o{ TaskFetched : emits
  TaskFetchConsumer }o--|| TaskQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TodoistPoller` | `application/TodoistPoller.java` |
| `TaskQueue` | `application/TaskQueue.java` |
| `TaskFetchConsumer` | `application/TaskFetchConsumer.java` |
| `TaskClassifierAgent` | `application/TaskClassifierAgent.java` |
| `OrganizerWorkflow` | `application/OrganizerWorkflow.java` |
| `TodoistTaskEntity` | `application/TodoistTaskEntity.java` (state in `domain/OrganizerTask.java`, events in `domain/OrganizerTaskEvent.java`) |
| `OrganizerView` | `application/OrganizerView.java` |
| `EvalRunner` | `application/EvalRunner.java` |
| `OrganizerEndpoint` | `api/OrganizerEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: classifier 15 s. On timeout, the workflow emits `TaskFailed` and terminates.
- **Guardrail gate**: `OrganizerWorkflow`'s `guardrailStep` is a synchronous in-workflow decision — no LLM call. It reads `ClassificationResult.confidence` and checks `targetProjectId` against a configured allow-list.
- **Idempotency**: every workflow uses `taskId` as the workflow id so duplicate fetch events fold into one workflow instance.
- **Eval sampling**: per tick, EvalRunner picks up to 5 UPDATED tasks with no `evalScore`, oldest-first.
