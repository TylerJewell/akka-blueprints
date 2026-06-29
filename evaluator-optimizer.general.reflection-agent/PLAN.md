# PLAN — reflection-agent

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

  Gen[GeneratorAgent]:::agent
  Ref[ReflectorAgent]:::agent

  WF[ReflectionWorkflow]:::wf
  TE[TaskEntity]:::ese
  SQ[SubmissionQueue]:::ese
  TV[TasksView]:::view
  SC[SubmissionConsumer]:::cons
  Sim[TaskSimulator]:::ta
  ES[EvalSampler]:::ta
  API[ReflectionEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit task| SQ
  Sim -.->|every 60s| SQ
  SQ -.->|subscribes| SC
  SC -->|start workflow| WF
  WF -->|generate / revise| Gen
  WF -->|reflect| Ref
  WF -->|emit events| TE
  TE -.->|projects| TV
  API -->|query / SSE| TV
  ES -.->|every 30s| TE
```

## Interaction sequence — J1 (convergence on iteration 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ReflectionEndpoint
  participant SQ as SubmissionQueue
  participant SC as SubmissionConsumer
  participant W as ReflectionWorkflow
  participant G as GeneratorAgent
  participant R as ReflectorAgent
  participant TE as TaskEntity
  participant TV as TasksView

  U->>API: POST /api/tasks {text}
  API->>SQ: append TaskSubmitted
  API-->>U: 202 {taskId}
  SQ->>SC: TaskSubmitted
  SC->>W: start({taskId, promptText, maxIterations=4})
  W->>TE: emit TaskCreated (GENERATING)

  W->>G: GENERATE(promptText)
  G-->>W: GeneratedResponse #1
  W->>TE: emit IterationGenerated (n=1)
  W->>TE: status REFLECTING
  W->>R: REFLECT(GeneratedResponse #1)
  R-->>W: Reflection{REVISE, score=3, 3 change requests}
  W->>TE: emit IterationReflected (n=1, REVISE)

  W->>G: REVISE_RESPONSE(promptText, prior, notes)
  G-->>W: GeneratedResponse #2
  W->>TE: emit IterationGenerated (n=2)
  W->>R: REFLECT(GeneratedResponse #2)
  R-->>W: Reflection{ACCEPT, score=5, rationale}
  W->>TE: emit IterationReflected (n=2, ACCEPT)
  W->>TE: emit TaskAccepted (n=2)
  TE-->>TV: project
  TV-->>U: SSE update
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> REFLECTING: IterationGenerated
  REFLECTING --> GENERATING: Reflection = REVISE, iterations < max
  REFLECTING --> ACCEPTED: Reflection = ACCEPT
  REFLECTING --> REJECTED_FINAL: Reflection = REVISE, iterations = max
  ACCEPTED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ TaskCreated : emits
  TaskEntity ||--o{ IterationGenerated : emits
  TaskEntity ||--o{ IterationReflected : emits
  TaskEntity ||--o{ TaskAccepted : emits
  TaskEntity ||--o{ TaskRejectedFinal : emits
  TaskEntity ||--o{ EvalRecorded : emits
  TasksView }o--|| TaskEntity : projects
  SubmissionQueue ||--o{ TaskSubmitted : emits
  SubmissionConsumer }o--|| SubmissionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `ReflectorAgent` | `application/ReflectorAgent.java` |
| `ReflectionTasks` | `application/ReflectionTasks.java` |
| `ReflectionWorkflow` | `application/ReflectionWorkflow.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `SubmissionQueue` | `application/SubmissionQueue.java` |
| `TasksView` | `application/TasksView.java` |
| `SubmissionConsumer` | `application/SubmissionConsumer.java` |
| `TaskSimulator` | `application/TaskSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `ReflectionEndpoint` | `api/ReflectionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `reflectStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `ReflectionEndpoint.submit` uses `(text, submittedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(taskId, iterationNumber)` so a tick that fires twice for the same iteration is a no-op on the entity side.
- **maxIterations ceiling:** read from `reflection-agent.loop.max-iterations` (default 4). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the best response and every critique on the entity.
- **No guardrail step:** unlike the content-editorial variant, this general-domain blueprint omits the deterministic length guardrail. The quality gate is fully delegated to `ReflectorAgent`.
