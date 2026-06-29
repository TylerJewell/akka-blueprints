# PLAN — fun-facts

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  API[FactEndpoint]:::ep
  Entity[FactRequestEntity]:::ese
  WF[FactRequestWorkflow]:::wf
  Agent[FactGeneratorAgent]:::agent
  View[FactRequestView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|generateStep runSingleTask| Agent
  Agent -->|FactCollection| WF
  WF -->|recordCollection| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as FactEndpoint
  participant E as FactRequestEntity
  participant W as FactRequestWorkflow
  participant A as FactGeneratorAgent

  U->>API: POST /api/fact-requests
  API->>E: submit(request)
  E-->>API: { requestId }
  API->>W: start(requestId)
  W->>E: markGenerating
  W->>A: runSingleTask(instructions: topic)
  A-->>W: FactCollection
  W->>E: recordCollection(collection)
  E-.->>U: SSE event(GENERATED)
```

## State machine — `FactRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> GENERATING: GenerationStarted
  GENERATING --> GENERATED: CollectionGenerated
  GENERATING --> FAILED: RequestFailed (agent error)
  PENDING --> FAILED: RequestFailed (validation error)
  GENERATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  FactRequestEntity ||--o{ FactRequested : emits
  FactRequestEntity ||--o{ GenerationStarted : emits
  FactRequestEntity ||--o{ CollectionGenerated : emits
  FactRequestEntity ||--o{ RequestFailed : emits
  FactRequestView }o--|| FactRequestEntity : projects
  FactRequestWorkflow }o--|| FactRequestEntity : reads-and-writes
  FactGeneratorAgent ||--o{ FactCollection : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `FactEndpoint` | `api/FactEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `FactRequestEntity` | `application/FactRequestEntity.java` (state in `domain/FactRequestState.java`, events in `domain/FactRequestEvent.java`) |
| `FactRequestWorkflow` | `application/FactRequestWorkflow.java` |
| `FactGeneratorAgent` | `application/FactGeneratorAgent.java` (tasks in `application/FactTasks.java`) |
| `FactRequestView` | `application/FactRequestView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `generateStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(FactRequestWorkflow::error)`. The 60 s on `generateStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"facts-" + requestId` as the workflow id; re-delivery of `FactRequested` is safe because `FactRequestEntity.submit` is version-guarded — a second submit against an already-initialized entity is a no-op.
- **One agent per request**: the AutonomousAgent instance id is `"generator-" + requestId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(2)` caps any retry at 2.
- **No saga / no compensation**: every step is either a single-task agent call or an entity write. There is nothing external to roll back.
- **No Consumer in this baseline**: the workflow starts directly from `FactEndpoint` after entity creation. There is no intermediate event-driven handoff — the smaller direct call is appropriate for a topic string that requires no pre-processing.
