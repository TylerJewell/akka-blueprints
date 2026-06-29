# PLAN — long-term-memory-agent

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

  API[ConversationEndpoint]:::ep
  App[AppEndpoint]:::ep
  SEntity[SessionEntity]:::ese
  MEntity[MemoryEntity]:::ese
  Sanitizer[MemorySanitizer]:::cons
  WF[MemoryExtractionWorkflow]:::wf
  Agent[MemoryAgent]:::agent
  SView[SessionView]:::view
  MView[MemoryView]:::view

  API -->|create session / add turn| SEntity
  API -->|start workflow| WF
  WF -->|recallStep query| MView
  WF -->|respondStep runSingleTask| Agent
  Agent -->|AgentReply| WF
  WF -->|recordReply + emitMemoryExtracted| SEntity
  SEntity -.->|MemoryExtracted| Sanitizer
  Sanitizer -->|storeMemory| MEntity
  SEntity -.->|projects| SView
  MEntity -.->|projects| MView
  API -->|list sessions / SSE| SView
  API -->|list memories| MView
  App -->|static| API
```

## Interaction sequence — J1 (happy path: first turn)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ConversationEndpoint
  participant SE as SessionEntity
  participant WF as MemoryExtractionWorkflow
  participant MV as MemoryView
  participant A as MemoryAgent
  participant MS as MemorySanitizer
  participant ME as MemoryEntity

  U->>API: POST /api/sessions/{id}/turns {message}
  API->>SE: addTurn(turnId, message)
  API->>WF: start(turnId, userId, message)
  WF->>MV: getMemoriesByUser(userId)
  MV-->>WF: [] (first session, empty)
  WF->>A: runSingleTask(message + memories.json attachment)
  A-->>WF: AgentReply{replyText, candidateMemories}
  WF->>SE: recordReply(turnId, replyText)
  WF->>SE: emitMemoryExtracted(turnId, candidateMemories)
  SE-.->>MS: MemoryExtracted event
  MS->>MS: redact PII from rawText
  MS->>ME: storeMemory(MemoryEntry)
  ME-.->>U: SSE event(MemoryStored)
  SE-.->>U: SSE event(ReplyRecorded)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> ACTIVE
  ACTIVE --> ACTIVE: TurnAdded / ReplyRecorded / MemoryExtracted
  ACTIVE --> CLOSING: max turns reached
  CLOSING --> CLOSED: SessionClosed
  ACTIVE --> CLOSED: SessionClosed (explicit close)
  ACTIVE --> ACTIVE: TurnFailed (turn fails; session stays open)
  CLOSED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ TurnAdded : emits
  SessionEntity ||--o{ ReplyRecorded : emits
  SessionEntity ||--o{ MemoryExtracted : emits
  SessionEntity ||--o{ SessionClosed : emits
  SessionEntity ||--o{ TurnFailed : emits
  MemoryEntity ||--o{ MemoryStored : emits
  MemoryEntity ||--o{ MemoryExpired : emits
  SessionView }o--|| SessionEntity : projects
  MemoryView }o--|| MemoryEntity : projects
  MemorySanitizer }o--|| SessionEntity : subscribes
  MemoryExtractionWorkflow }o--|| SessionEntity : reads-and-writes
  MemoryExtractionWorkflow }o--|| MemoryView : reads
  MemoryAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversationEndpoint` | `api/ConversationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `MemoryEntity` | `application/MemoryEntity.java` (state in `domain/UserMemoryStore.java`, events in `domain/MemoryEvent.java`) |
| `MemorySanitizer` | `application/MemorySanitizer.java` |
| `MemoryExtractionWorkflow` | `application/MemoryExtractionWorkflow.java` |
| `MemoryAgent` | `application/MemoryAgent.java` (tasks in `application/MemoryTasks.java`) |
| `SessionView` | `application/SessionView.java` |
| `MemoryView` | `application/MemoryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `recallStep` 5 s, `respondStep` 60 s, `extractStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(MemoryExtractionWorkflow::error)`. The 60 s on `respondStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"turn-" + turnId` as the workflow id; the `MemorySanitizer` Consumer is allowed to redeliver `MemoryExtracted` events because `MemoryEntity.storeMemory` is guarded by `entryId` uniqueness — a duplicate write for the same entryId is a no-op.
- **One agent per user**: the AutonomousAgent instance id is `"agent-" + userId`, giving each user their own conversation context and iteration budget. The agent's `capability(...).maxIterationsPerTask(2)` limits retries.
- **Memory recall is read-only**: `recallStep` queries `MemoryView` — a projection, not the entity directly. The view is eventually consistent; on the very first turn the view is empty, which is a valid and handled case.
- **Sanitizer is asynchronous**: the `extractStep` completes as soon as `MemoryExtracted` events are emitted onto `SessionEntity`. The `MemorySanitizer` Consumer processes them asynchronously. The UI receives a `MemoryStored` SSE event when persistence completes. There is no saga / no compensation — memory writes are append-only.
- **Session cap**: `SessionEntity.addTurn` rejects if `turns.size() >= 50`. This keeps entity state bounded. A deployer who needs longer conversations creates a new session; prior sessions' memories remain accessible via `MemoryEntity`.
