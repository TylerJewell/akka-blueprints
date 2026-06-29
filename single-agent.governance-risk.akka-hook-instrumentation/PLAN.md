# PLAN — hook-instrumentation

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

  API[ObservationEndpoint]:::ep
  Entity[ObservationEntity]:::ese
  Consumer[HookLogConsumer]:::cons
  WF[ObservationWorkflow]:::wf
  Agent[ActivityObserverAgent]:::agent
  G1[BeforeToolCallGuardrail]:::guard
  G2[AfterToolCallGuardrail]:::guard
  G3[BeforeLlmCallGuardrail]:::guard
  Scorer[CoverageScorer]:::guard
  View[ObservationView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|TaskSubmitted| Consumer
  Consumer -->|initHookChain| Entity
  Consumer -->|start workflow| WF
  WF -->|hookInitStep| Entity
  WF -->|executeStep runSingleTask| Agent
  Agent -.->|before-tool-call| G1
  Agent -.->|after-tool-call| G2
  Agent -.->|before-llm-call| G3
  Agent -->|AgentOutcome| WF
  WF -->|recordOutcome| Entity
  WF -->|coverageStep score| Scorer
  Scorer -->|CoverageScore| WF
  WF -->|recordCoverage| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ObservationEndpoint
  participant E as ObservationEntity
  participant C as HookLogConsumer
  participant W as ObservationWorkflow
  participant A as ActivityObserverAgent
  participant G1 as BeforeToolCallGuardrail
  participant G2 as AfterToolCallGuardrail
  participant G3 as BeforeLlmCallGuardrail
  participant Sc as CoverageScorer

  U->>API: POST /api/observations
  API->>E: submit(request)
  E-->>API: { observationId }
  E-.->>C: TaskSubmitted
  C->>E: initHookChain(hookConfig)
  C->>W: start(observationId)
  W->>E: hookInitStep — HookChainInitialized
  W->>E: markExecuting
  W->>A: runSingleTask(taskDef)
  A->>G3: before-llm-call(prompt)
  G3-->>A: PASS_THROUGH / scrubbed prompt
  A->>G1: before-tool-call(tool, input)
  G1-->>A: ALLOWED
  A->>A: execute tool
  A->>G2: after-tool-call(tool, output)
  G2-->>A: PASS_THROUGH / sanitized output
  A-->>W: AgentOutcome
  W->>E: recordOutcome(outcome)
  W->>Sc: score(outcome, request)
  Sc-->>W: CoverageScore
  W->>E: recordCoverage(coverage)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `ObservationEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> HOOK_CHAIN_READY: HookChainInitialized
  HOOK_CHAIN_READY --> EXECUTING: ExecutionStarted
  EXECUTING --> COMPLETED: OutcomeRecorded + CoverageScored
  EXECUTING --> FAILED: ObservationFailed (agent or guardrail error)
  SUBMITTED --> FAILED: ObservationFailed (hook init error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ObservationEntity ||--o{ TaskSubmitted : emits
  ObservationEntity ||--o{ HookChainInitialized : emits
  ObservationEntity ||--o{ ExecutionStarted : emits
  ObservationEntity ||--o{ OutcomeRecorded : emits
  ObservationEntity ||--o{ CoverageScored : emits
  ObservationEntity ||--o{ ObservationFailed : emits
  ObservationView }o--|| ObservationEntity : projects
  HookLogConsumer }o--|| ObservationEntity : subscribes
  ObservationWorkflow }o--|| ObservationEntity : reads-and-writes
  ActivityObserverAgent ||--o{ AgentOutcome : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ObservationEndpoint` | `api/ObservationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ObservationEntity` | `application/ObservationEntity.java` (state in `domain/Observation.java`, events in `domain/ObservationEvent.java`) |
| `HookLogConsumer` | `application/HookLogConsumer.java` |
| `ObservationWorkflow` | `application/ObservationWorkflow.java` |
| `ActivityObserverAgent` | `application/ActivityObserverAgent.java` (tasks in `application/ObservationTasks.java`) |
| `BeforeToolCallGuardrail` | `application/BeforeToolCallGuardrail.java` |
| `AfterToolCallGuardrail` | `application/AfterToolCallGuardrail.java` |
| `BeforeLlmCallGuardrail` | `application/BeforeLlmCallGuardrail.java` |
| `CoverageScorer` | `application/CoverageScorer.java` |
| `ObservationView` | `application/ObservationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `hookInitStep` 10 s, `executeStep` 90 s, `coverageStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ObservationWorkflow::error)`. The 90 s on `executeStep` accommodates multi-tool LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"obs-" + observationId` as the workflow id. The `HookLogConsumer` Consumer may redeliver `TaskSubmitted` events; `ObservationEntity.initHookChain` is event-version-guarded — a second init against an already-initialized observation is a no-op.
- **One agent per task**: the AutonomousAgent instance id is `"observer-" + observationId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(5)` bounds the hook-triggered retry chain.
- **Guardrail accumulation**: all three guardrails append entries to the task's shared hook log context. When the agent returns `AgentOutcome`, the `hookLog` field carries the full accumulated list from all guardrails across all iterations.
- **Coverage is synchronous and deterministic**: `CoverageScorer` runs in-process inside `coverageStep`. No LLM call — the same outcome always scores the same. This preserves the single-agent invariant.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. Nothing external to roll back.
