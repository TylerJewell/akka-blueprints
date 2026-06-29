# PLAN — inline-agent-demo

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

  API[RunEndpoint]:::ep
  Entity[RunEntity]:::ese
  WF[RunWorkflow]:::wf
  Agent[InlineAgentRunner]:::agent
  View[RunView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|validateStep read| Entity
  WF -->|runStep markRunning| Entity
  WF -->|runStep runSingleTask| Agent
  Agent -->|AgentResponse| WF
  WF -->|complete| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as RunEndpoint
  participant E as RunEntity
  participant W as RunWorkflow
  participant A as InlineAgentRunner

  U->>API: POST /api/runs {question, agentDefinition}
  API->>E: submit(request)
  E-->>API: { runId }
  API->>W: start(runId)
  W->>E: getRun (validateStep)
  E-->>W: run.request.isPresent()
  W->>E: markRunning (RunStarted)
  W->>A: runSingleTask(AgentDefinition + question)
  A-->>W: AgentResponse
  W->>E: complete(response)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `RunEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> RUNNING: RunStarted
  RUNNING --> COMPLETED: RunCompleted
  RUNNING --> FAILED: RunFailed (agent error)
  RECEIVED --> FAILED: RunFailed (validation error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  RunEntity ||--o{ RunReceived : emits
  RunEntity ||--o{ RunStarted : emits
  RunEntity ||--o{ RunCompleted : emits
  RunEntity ||--o{ RunFailed : emits
  RunView }o--|| RunEntity : projects
  RunWorkflow }o--|| RunEntity : reads-and-writes
  InlineAgentRunner ||--o{ AgentResponse : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RunEndpoint` | `api/RunEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `RunEntity` | `application/RunEntity.java` (state in `domain/Run.java`, events in `domain/RunEvent.java`) |
| `RunWorkflow` | `application/RunWorkflow.java` |
| `InlineAgentRunner` | `application/InlineAgentRunner.java` (tasks in `application/RunTasks.java`) |
| `RunView` | `application/RunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `validateStep` 5 s, `runStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(RunWorkflow::error)`. The 60 s on `runStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"run-" + runId` as the workflow id. The `RunEndpoint` starts the workflow only after the entity write succeeds; a duplicate submission on the same `runId` hits an already-running workflow, which is a no-op.
- **One agent per run**: the AutonomousAgent instance id is `"runner-" + runId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps internal retries.
- **Inline agent construction**: the `AgentDefinition` object passed to `InlineAgentRunner` is built inside `runStep` from the `InlineAgentRequest` stored on the entity. The base system prompt from `prompts/inline-agent-runner.md` is loaded at startup and prepended to the caller-supplied `systemPrompt` at task time.
- **No saga / no compensation**: the only external call is the single LLM task. There is nothing external to roll back.
- **Validation is synchronous**: `validateStep` reads the entity and checks fields in-process. Malformed requests fail fast without ever reaching the LLM.
