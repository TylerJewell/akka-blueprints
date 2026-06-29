# PLAN — mem0-react-agent

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
  classDef monitor fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[AgentEndpoint]:::ep
  SessionEnt[SessionEntity]:::ese
  MemoryEnt[MemoryEntity]:::ese
  Sanitizer[FactSanitizer]:::cons
  Drift[DriftMonitor]:::monitor
  WF[MemoryWriteWorkflow]:::wf
  Agent[Mem0ReactAgent]:::agent
  SessionV[SessionView]:::view
  MemoryV[MemoryView]:::view
  App[AppEndpoint]:::ep

  API -->|open/startTurn| SessionEnt
  API -->|runSingleTask| Agent
  Agent -->|store-memory tool| MemoryEnt
  Agent -->|recall-memories tool| MemoryV
  Agent -->|AgentAnswer| SessionEnt
  MemoryEnt -.->|FactStoreRequested| Sanitizer
  Sanitizer -->|attachSanitized| MemoryEnt
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| MemoryEnt
  WF -->|persistStep| MemoryEnt
  MemoryEnt -.->|FactPersisted| Drift
  Drift -->|MemoryDriftSignaled| MemoryV
  SessionEnt -.->|projects| SessionV
  MemoryEnt -.->|projects| MemoryV
  API -->|list/SSE| SessionV
  API -->|list/SSE| MemoryV
  App -->|static| API
```

## Interaction sequence — J1 (happy path: user sends message, fact stored, recalled next session)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as AgentEndpoint
  participant SE as SessionEntity
  participant A as Mem0ReactAgent
  participant ME as MemoryEntity
  participant FS as FactSanitizer
  participant W as MemoryWriteWorkflow
  participant MV as MemoryView

  U->>API: POST /api/sessions/{id}/turns
  API->>SE: startTurn(turnId, text)
  API->>A: runSingleTask(ANSWER_TURN)
  A->>MV: recall-memories(userId)
  MV-->>A: [] (no facts yet)
  A->>A: reason, answer
  A->>ME: store-memory tool → requestStore(fact)
  ME-.->>FS: FactStoreRequested
  FS->>FS: redact PII
  FS->>ME: attachSanitized
  FS->>W: start(factId)
  W->>ME: poll getMemory
  ME-->>W: sanitized.isPresent()
  W->>ME: persist(factId)
  ME-.->>MV: FactPersisted
  A-->>API: AgentAnswer
  API->>SE: recordAnswer(answer)
  SE-.->>U: SSE event(TurnAnswered)

  Note over U,MV: Next session
  U->>API: POST /api/sessions/{id2}/turns
  API->>A: runSingleTask(ANSWER_TURN)
  A->>MV: recall-memories(userId)
  MV-->>A: [stored fact]
  A->>A: weave fact into answer
  A-->>API: AgentAnswer (uses recalled fact)
```

## State machine — `MemoryEntity`

```mermaid
stateDiagram-v2
  [*] --> SANITIZE_PENDING
  SANITIZE_PENDING --> SANITIZED: FactSanitized
  SANITIZED --> PERSISTED: FactPersisted
  SANITIZE_PENDING --> FAILED: FactFailed (sanitizer error)
  SANITIZED --> FAILED: FactFailed (persist error)
  PERSISTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionOpened : emits
  SessionEntity ||--o{ TurnStarted : emits
  SessionEntity ||--o{ TurnAnswered : emits
  SessionEntity ||--o{ SessionClosed : emits
  MemoryEntity ||--o{ FactStoreRequested : emits
  MemoryEntity ||--o{ FactSanitized : emits
  MemoryEntity ||--o{ FactPersisted : emits
  MemoryEntity ||--o{ FactFailed : emits
  SessionView }o--|| SessionEntity : projects
  MemoryView }o--|| MemoryEntity : projects
  FactSanitizer }o--|| MemoryEntity : subscribes
  DriftMonitor }o--|| MemoryEntity : subscribes
  MemoryWriteWorkflow }o--|| MemoryEntity : reads-and-writes
  Mem0ReactAgent ||--o{ AgentAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AgentEndpoint` | `api/AgentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `MemoryEntity` | `application/MemoryEntity.java` (state in `domain/Memory.java`, events in `domain/MemoryEvent.java`) |
| `FactSanitizer` | `application/FactSanitizer.java` |
| `MemoryWriteWorkflow` | `application/MemoryWriteWorkflow.java` |
| `DriftMonitor` | `application/DriftMonitor.java` |
| `Mem0ReactAgent` | `application/Mem0ReactAgent.java` (tasks in `application/AgentTasks.java`) |
| `SessionView` | `application/SessionView.java` |
| `MemoryView` | `application/MemoryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `persistStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(MemoryWriteWorkflow::error)`. The 10 s on `persistStep` is generous for an in-process write (Lesson 4).
- **Agent turn timeout**: `ANSWER_TURN` task uses `maxIterationsPerTask(8)`, accommodating up to 8 ReAct reasoning steps including tool calls within one turn.
- **Idempotency**: `MemoryWriteWorkflow` uses `"mem-" + factId` as its id; `FactSanitizer` is allowed to redeliver because `MemoryEntity.attachSanitized` is version-guarded — a second sanitize against an already-sanitized fact is a no-op.
- **DriftMonitor is eventually consistent**: the monitor reads `MemoryView.countByUserId` after each `FactPersisted` event; the view may lag by one event in high-throughput bursts. The threshold is advisory, not a hard cap.
- **One agent per session**: the AutonomousAgent instance id is `"agent-" + sessionId`. Each session gets its own conversation context; facts are shared across sessions via `MemoryView`.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
