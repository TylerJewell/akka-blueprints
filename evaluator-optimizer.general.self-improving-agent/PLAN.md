# PLAN — self-improving-agent

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

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

  Executor[ExecutorAgent]:::agent
  Optimizer[OptimizerAgent]:::agent

  WF[ImprovementWorkflow]:::wf
  Config[AgentConfigEntity]:::ese
  Queue[TaskQueueEntity]:::ese
  View[ConfigView]:::view
  Consumer[TaskResultConsumer]:::cons
  Sim[TaskSimulator]:::ta
  Sampler[AttestationSampler]:::ta
  API[AgentEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit task| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|execute / regression| Executor
  WF -->|propose revision| Optimizer
  WF -->|emit events| Config
  Config -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 30s| Config
```

## Interaction sequence — J1 (revision applied on cycle 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as AgentEndpoint
  participant Q as TaskQueueEntity
  participant C as TaskResultConsumer
  participant W as ImprovementWorkflow
  participant X as ExecutorAgent
  participant O as OptimizerAgent
  participant E as AgentConfigEntity
  participant V as ConfigView

  U->>API: POST /api/tasks {instruction, expectedHint}
  API->>Q: append TaskSubmitted
  API-->>U: 202 {taskId}
  Q->>C: TaskSubmitted (batch complete)
  C->>W: start({configId, tasks, maxRevisions=3})
  W->>E: emit ConfigInitialized (IMPROVING)

  W->>X: EXECUTE_TASK(instruction, currentPrompt)
  X-->>W: ExecutionResult{qualityScore=3, latencyMs=210}
  Note over W: evaluateStep — builds PerformanceRecord
  W->>O: PROPOSE_REVISION(performanceRecord)
  O-->>W: PromptRevision{proposedPrompt, rationale}
  W->>E: emit RevisionProposed (cycle=1)

  Note over W: attestStep — re-runs ExecutorAgent on reference set
  W->>X: REGRESSION_EXECUTE(refTask1, proposedPrompt)
  X-->>W: ExecutionResult{qualityScore=3}
  W->>E: emit AttestationRecorded (FAILED, delta=-0.1, cycle=1)

  W->>O: PROPOSE_REVISION(performanceRecord + failedAttestation)
  O-->>W: PromptRevision{proposedPrompt v2, rationale}
  W->>E: emit RevisionProposed (cycle=2)
  W->>X: REGRESSION_EXECUTE(refTask1, proposedPrompt v2)
  X-->>W: ExecutionResult{qualityScore=5}
  W->>E: emit AttestationRecorded (PASSED, delta=+0.8, cycle=2)
  W->>E: emit RevisionApplied (APPLIED)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `AgentConfigEntity`

```mermaid
stateDiagram-v2
  [*] --> ACTIVE
  ACTIVE --> IMPROVING: ImprovementWorkflow starts
  IMPROVING --> IMPROVING: AttestationFailed, revisions < max
  IMPROVING --> APPLIED: AttestationPassed
  IMPROVING --> REVISION_REJECTED: AttestationFailed, revisions = max
  APPLIED --> [*]
  REVISION_REJECTED --> [*]
```

## Entity model

```mermaid
erDiagram
  AgentConfigEntity ||--o{ ConfigInitialized : emits
  AgentConfigEntity ||--o{ RevisionProposed : emits
  AgentConfigEntity ||--o{ AttestationRecorded : emits
  AgentConfigEntity ||--o{ RevisionApplied : emits
  AgentConfigEntity ||--o{ RevisionRejected : emits
  AgentConfigEntity ||--o{ EvalRecorded : emits
  ConfigView }o--|| AgentConfigEntity : projects
  TaskQueueEntity ||--o{ TaskSubmitted : emits
  TaskResultConsumer }o--|| TaskQueueEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ExecutorAgent` | `application/ExecutorAgent.java` |
| `OptimizerAgent` | `application/OptimizerAgent.java` |
| `AgentTasks` | `application/AgentTasks.java` |
| `ImprovementWorkflow` | `application/ImprovementWorkflow.java` |
| `AgentConfigEntity` | `application/AgentConfigEntity.java` (state in `domain/AgentConfig.java`, events in `domain/ConfigEvent.java`) |
| `TaskQueueEntity` | `application/TaskQueueEntity.java` |
| `ConfigView` | `application/ConfigView.java` |
| `TaskResultConsumer` | `application/TaskResultConsumer.java` |
| `TaskSimulator` | `application/TaskSimulator.java` |
| `AttestationSampler` | `application/AttestationSampler.java` |
| `AgentEndpoint` | `api/AgentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `executeStep`, `proposeStep`, and `attestStep` each carry `stepTimeout(Duration.ofSeconds(90))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REVISION_REJECTED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `AgentEndpoint.submitTask` uses `(instruction, submittedBy)` over a 10 s window as the dedup key.
- **AttestationSampler idempotency:** the sampler keys its `recordEval` calls on `(configId, cycleNumber)` so a tick that fires twice for the same cycle is a no-op on the entity side.
- **maxRevisions ceiling:** read from `self-improving-agent.improvement.max-revisions` (default 3). The workflow checks the count BEFORE calling `proposeStep` for the next iteration; it never recurses past the ceiling.
- **Attestation determinism:** the reference task set used in `attestStep` is fixed at workflow start (first 3 tasks from `sample-events/tasks.jsonl`). The baseline quality score is computed from the first `executeStep` run; the delta is `meanRegressionScore - baselineScore`.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`REVISION_REJECTED`) preserves the best-scoring proposed revision and every attestation verdict on the entity.
- **Monitor SSE:** `/api/configs/monitor/sse` streams only `RevisionApplied` and `RevisionRejected` events, filtered from the full `AgentConfigEntity` event stream. Clients connect once and receive all future terminal transitions.
