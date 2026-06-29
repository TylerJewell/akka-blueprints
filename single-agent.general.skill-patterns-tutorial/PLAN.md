# PLAN — skill-patterns-tutorial

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
  classDef stub fill:#251503,stroke:#F97316,color:#F97316;

  API[SkillRunEndpoint]:::ep
  Entity[SkillRunEntity]:::ese
  WF[SkillRunWorkflow]:::wf
  Agent[SkillDemoAgent]:::agent
  Stub[SkillToolStub]:::stub
  View[SkillRunView]:::view
  App[AppEndpoint]:::ep

  API -->|request| Entity
  API -->|start workflow| WF
  WF -->|runStep runSingleTask| Agent
  Agent -.->|tool call| Stub
  Stub -.->|tool response| Agent
  Agent -->|SkillResult| WF
  WF -->|recordResult| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, any pattern)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SkillRunEndpoint
  participant E as SkillRunEntity
  participant W as SkillRunWorkflow
  participant A as SkillDemoAgent
  participant St as SkillToolStub

  U->>API: POST /api/skill-runs {pattern, input}
  API->>E: request(runRequest)
  E-->>API: { runId }
  API->>W: start(runId)
  W->>E: markRunning
  W->>A: runSingleTask(task text)
  alt EXTERNAL pattern
    A->>St: GET /api/tool-stub/lookup?key=...
    St-->>A: { key, value, source }
  end
  A-->>W: SkillResult
  W->>E: recordResult(result)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `SkillRunEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> RUNNING: SkillRunStarted
  RUNNING --> COMPLETED: SkillRunCompleted
  RUNNING --> FAILED: SkillRunFailed (agent or timeout)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SkillRunEntity ||--o{ SkillRunRequested : emits
  SkillRunEntity ||--o{ SkillRunStarted : emits
  SkillRunEntity ||--o{ SkillRunCompleted : emits
  SkillRunEntity ||--o{ SkillRunFailed : emits
  SkillRunView }o--|| SkillRunEntity : projects
  SkillRunWorkflow }o--|| SkillRunEntity : reads-and-writes
  SkillDemoAgent ||--o{ SkillResult : returns
  SkillDemoAgent }o--|| SkillToolStub : calls-via-tool
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SkillRunEndpoint` | `api/SkillRunEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SkillToolStub` | `api/SkillToolStub.java` |
| `SkillRunEntity` | `application/SkillRunEntity.java` (state in `domain/SkillRun.java`, events in `domain/SkillRunEvent.java`) |
| `SkillRunWorkflow` | `application/SkillRunWorkflow.java` |
| `SkillDemoAgent` | `application/SkillDemoAgent.java` (tasks in `application/SkillDemoTasks.java`) |
| `SkillRunView` | `application/SkillRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `runStep` 90 s, `recordStep` 10 s, `errorStep` 5 s. Default step recovery `maxRetries(1).failoverTo(SkillRunWorkflow::errorStep)`. The 90 s on `runStep` accommodates LLM latency for the more open-ended meta-creator path (Lesson 4).
- **Idempotency**: every workflow uses `"skill-run-" + runId` as the workflow id; re-delivering a `SkillRunRequested` event to the entity is a no-op because the entity guards against double-request on a non-empty run.
- **One agent per run**: the AutonomousAgent instance id is `"skill-agent-" + runId`, scoping each task's conversation context to the invocation.
- **Concurrent pattern invocations**: all four patterns run through the same `SkillDemoAgent` class but distinct instance ids, so concurrent runs for different patterns do not share state.
- **In-process tool stub**: `SkillToolStub` is a standard `HttpEndpoint` registered in Bootstrap. The external-skill pattern's `ToolDefinition.http(...)` resolves against `http://localhost:9588/api/tool-stub/lookup` — no network hop needed.
- **Dynamic skill registry**: registered `SkillDefinition` objects live in an `AtomicReference<List<SkillDefinition>>` on `SkillRunEndpoint`. They are not replicated and reset on restart. This is intentional for a tutorial sample — a production deployer would persist them to an entity.
