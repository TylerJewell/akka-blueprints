# PLAN — multi-provider-router

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
  classDef agg fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[RouterEndpoint]:::ep
  Entity[CallEntity]:::ese
  Monitor[PerformanceMonitor]:::cons
  WF[RoutingWorkflow]:::wf
  Agent[RouterAgent]:::agent
  Agg[EvaluationAggregator]:::agg
  CallView[CallView]:::view
  MonView[MonitorView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|dispatchStep runSingleTask| Agent
  Agent -->|CallResult| WF
  WF -->|complete| Entity
  Entity -.->|CallCompleted| Monitor
  Monitor -->|window threshold| Agg
  Agg -->|EvalReport| MonView
  Entity -.->|projects| CallView
  API -->|list/SSE| CallView
  API -->|monitor| MonView
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as RouterEndpoint
  participant E as CallEntity
  participant W as RoutingWorkflow
  participant A as RouterAgent
  participant M as PerformanceMonitor
  participant Agg as EvaluationAggregator
  participant MV as MonitorView

  U->>API: POST /api/calls
  API->>E: submit(request)
  E-->>API: { callId }
  API->>W: start(callId)
  W->>E: dispatch
  W->>A: runSingleTask(promptText)
  A-->>W: CallResult
  W->>E: complete(result)
  E-.->>M: CallCompleted
  alt window threshold reached
    M->>Agg: aggregate(window calls)
    Agg-->>M: List<ProviderStats>
    M->>MV: upsert(EvalReport)
  end
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `CallEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> DISPATCHED: CallDispatched
  DISPATCHED --> COMPLETED: CallCompleted
  DISPATCHED --> FAILED: CallFailed (timeout / provider error)
  PENDING --> FAILED: CallFailed (workflow start error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  CallEntity ||--o{ CallSubmitted : emits
  CallEntity ||--o{ CallDispatched : emits
  CallEntity ||--o{ CallCompleted : emits
  CallEntity ||--o{ CallFailed : emits
  CallView }o--|| CallEntity : projects
  MonitorView }o--|| PerformanceMonitor : receives
  PerformanceMonitor }o--|| CallEntity : subscribes
  RoutingWorkflow }o--|| CallEntity : reads-and-writes
  RouterAgent ||--o{ CallResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RouterEndpoint` | `api/RouterEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CallEntity` | `application/CallEntity.java` (state in `domain/CallRecord.java`, events in `domain/CallEvent.java`) |
| `RoutingWorkflow` | `application/RoutingWorkflow.java` |
| `RouterAgent` | `application/RouterAgent.java` (tasks in `application/RouterTasks.java`) |
| `PerformanceMonitor` | `application/PerformanceMonitor.java` |
| `EvaluationAggregator` | `application/EvaluationAggregator.java` |
| `CallView` | `application/CallView.java` |
| `MonitorView` | `application/MonitorView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `dispatchStep` 60 s, `recordStep` 5 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(RoutingWorkflow::error)`. The 60 s on `dispatchStep` accommodates provider latency variance across the two backends (Lesson 4).
- **Idempotency**: every workflow uses `"routing-" + callId` as the workflow id; the `RouterEndpoint` is responsible for minting unique callIds. A duplicate POST with the same callId is a client error (400).
- **One agent per call**: the AutonomousAgent instance id is `"router-" + callId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(2)` bounds any retry on malformed output.
- **Provider selection**: the LiteLLMRouterModel shuffle strategy picks a backend per call without locking. The RoutingWorkflow does not control which backend is selected — that is the model-provider's concern. A ProviderHint of OPENAI or ANTHROPIC pins the call; AUTO lets the router decide.
- **Eval is asynchronous and deterministic**: `EvaluationAggregator` runs in-process inside `PerformanceMonitor` after the window threshold is crossed. No LLM call, no external service — the same set of call records always produces the same stats. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. Nothing external to roll back.
