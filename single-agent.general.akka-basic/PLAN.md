# PLAN — akka-basic

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
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[PromptEndpoint]:::ep
  Entity[PromptEntity]:::ese
  WF[PromptWorkflow]:::wf
  Agent[PromptAgent]:::agent
  Guard[OutputGuardrail]:::guard
  View[PromptView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|markProcessing| Entity
  WF -->|processStep runSingleTask| Agent
  Agent -.->|after-agent-response| Guard
  Agent -->|PromptResult| WF
  WF -->|recordResult| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as PromptEndpoint
  participant E as PromptEntity
  participant W as PromptWorkflow
  participant A as PromptAgent
  participant G as OutputGuardrail

  U->>API: POST /api/runs
  API->>E: submit(request)
  E-->>API: { runId }
  API->>W: start(runId)
  W->>E: markProcessing
  W->>A: runSingleTask(prompt + categoryHint)
  A->>G: after-agent-response(candidate)
  G-->>A: accept
  A-->>W: PromptResult
  W->>E: recordResult(result)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `PromptEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PROCESSING: RunStarted
  PROCESSING --> COMPLETED: ResultRecorded
  PROCESSING --> FAILED: RunFailed (agent error or guardrail exhaustion)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PromptEntity ||--o{ RunSubmitted : emits
  PromptEntity ||--o{ RunStarted : emits
  PromptEntity ||--o{ ResultRecorded : emits
  PromptEntity ||--o{ RunFailed : emits
  PromptView }o--|| PromptEntity : projects
  PromptWorkflow }o--|| PromptEntity : reads-and-writes
  PromptAgent ||--o{ PromptResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PromptEndpoint` | `api/PromptEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PromptEntity` | `application/PromptEntity.java` (state in `domain/Run.java`, events in `domain/RunEvent.java`) |
| `PromptWorkflow` | `application/PromptWorkflow.java` |
| `PromptAgent` | `application/PromptAgent.java` (tasks in `application/PromptTasks.java`) |
| `OutputGuardrail` | `application/OutputGuardrail.java` |
| `PromptView` | `application/PromptView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `processStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(PromptWorkflow::error)`. The 60 s on `processStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"prompt-" + runId` as the workflow id; the endpoint is the sole entry point so double-submission is caught by entity command validation before the workflow starts.
- **One agent per run**: the AutonomousAgent instance id is `"agent-" + runId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `OutputGuardrail` rejects a candidate response the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, `processStep` fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: `processStep` is a single task call. There is nothing external to roll back.
