# PLAN — react-loop-agent

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

  API[RunEndpoint]:::ep
  Entity[RunEntity]:::ese
  Dispatcher[ToolDispatcher]:::cons
  WF[ReActWorkflow]:::wf
  Agent[ReActAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  Evaluator[ChainEvaluator]:::guard
  View[RunView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|initStep markRunning| Entity
  WF -->|loopStep runSingleTask| Agent
  Agent -->|Step| WF
  WF -->|check action| Guard
  Guard -->|allow / block| WF
  WF -->|recordStep / blockAction| Entity
  Entity -.->|ActionRequested| Dispatcher
  Dispatcher -->|tool stub| Dispatcher
  Dispatcher -->|observationStep| Entity
  WF -->|finalizeStep complete| Entity
  WF -->|finalizeStep score| Evaluator
  Evaluator -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
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
  participant W as ReActWorkflow
  participant A as ReActAgent
  participant G as ToolCallGuardrail
  participant D as ToolDispatcher
  participant Ch as ChainEvaluator

  U->>API: POST /api/runs
  API->>E: submit(request)
  E-->>API: { runId }
  API->>W: start(runId)
  W->>E: markRunning
  loop ReAct iterations
    W->>A: runSingleTask(query + tools)
    A-->>W: Step(THOUGHT)
    W->>E: recordStep(thought)
    W->>A: continue
    A-->>W: Step(ACTION)
    W->>G: check(action, request)
    G-->>W: allowed
    W->>E: recordStep(action)
    E-.->>D: ActionRequested
    D->>D: execute tool
    D->>E: recordStep(observation)
    W->>A: continue
    A-->>W: Step(ANSWER)
  end
  W->>E: complete(result)
  W->>Ch: score(steps, result)
  Ch-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `RunEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> RUNNING: RunStarted
  RUNNING --> COMPLETED: RunCompleted
  RUNNING --> EXHAUSTED: RunExhausted (budget reached)
  RUNNING --> FAILED: RunFailed (workflow error)
  COMPLETED --> COMPLETED: EvaluationScored
  EXHAUSTED --> EXHAUSTED: EvaluationScored
  COMPLETED --> [*]
  EXHAUSTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  RunEntity ||--o{ RunSubmitted : emits
  RunEntity ||--o{ RunStarted : emits
  RunEntity ||--o{ StepRecorded : emits
  RunEntity ||--o{ ActionBlocked : emits
  RunEntity ||--o{ RunCompleted : emits
  RunEntity ||--o{ RunExhausted : emits
  RunEntity ||--o{ EvaluationScored : emits
  RunEntity ||--o{ RunFailed : emits
  RunView }o--|| RunEntity : projects
  ToolDispatcher }o--|| RunEntity : subscribes
  ReActWorkflow }o--|| RunEntity : reads-and-writes
  ReActAgent ||--o{ ReActResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RunEndpoint` | `api/RunEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `RunEntity` | `application/RunEntity.java` (state in `domain/Run.java`, events in `domain/RunEvent.java`) |
| `ToolDispatcher` | `application/ToolDispatcher.java` |
| `ReActWorkflow` | `application/ReActWorkflow.java` |
| `ReActAgent` | `application/ReActAgent.java` (tasks in `application/RunTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `ChainEvaluator` | `application/ChainEvaluator.java` |
| `RunView` | `application/RunView.java` |
| `CalculatorTool` | `application/tools/CalculatorTool.java` |
| `WebLookupTool` | `application/tools/WebLookupTool.java` |
| `DataFetchTool` | `application/tools/DataFetchTool.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `initStep` 5 s, `loopStep` 60 s, `finalizeStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ReActWorkflow::error)`. The 60 s on `loopStep` accommodates LLM latency per ReAct iteration (Lesson 4).
- **Idempotency**: every workflow uses `"run-" + runId` as the workflow id. `RunEntity.markRunning` is event-version-guarded — a redelivered `RunStarted` against an already-running entity is a no-op.
- **One agent per run**: the AutonomousAgent instance id is `"agent-" + runId`, giving each run its own conversation context. `maxIterationsPerTask(10)` caps the ReAct loop.
- **Tool guardrail is not an agent hook**: `ToolCallGuardrail` is called synchronously by `loopStep` before forwarding an ACTION to `ToolDispatcher`. A blocked call produces a synthetic OBSERVATION step that the agent receives on its next iteration. The block is recorded as `ActionBlocked` on the entity — the audit trail is complete regardless of whether the agent recovers.
- **Eval is synchronous and deterministic**: `ChainEvaluator` runs in-process inside `finalizeStep`. No LLM call — the same step trace always scores the same. This maintains the single-agent guarantee.
- **Terminal states are COMPLETED, EXHAUSTED, FAILED**: `EvaluationScored` is appended to a COMPLETED or EXHAUSTED run without changing the terminal status. A FAILED run does not receive an eval — the partial trace is preserved for inspection.
