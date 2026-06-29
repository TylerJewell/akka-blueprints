# PLAN — memory-bank

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

  API[MemoryEndpoint]:::ep
  Entity[MemoryEntity]:::ese
  Sanitizer[MemorySanitizer]:::cons
  WF[MemoryWorkflow]:::wf
  Agent[MemoryAgent]:::agent
  View[MemoryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|MemoryEntrySubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|storeStep runSingleTask| Agent
  Agent -->|StoreConfirmation or RecallResult| WF
  WF -->|recordResult| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path: REMEMBER)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as MemoryEndpoint
  participant E as MemoryEntity
  participant S as MemorySanitizer
  participant W as MemoryWorkflow
  participant A as MemoryAgent

  U->>API: POST /api/memories {operation: REMEMBER}
  API->>E: submit(request)
  E-->>API: { memoryId }
  E-.->>S: MemoryEntrySubmitted
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(memoryId)
  W->>E: poll getMemory
  E-->>W: sanitized.isPresent()
  W->>E: markStoring
  W->>A: runSingleTask(REMEMBER_CONTENT, sanitizedContent + namespace)
  A-->>W: StoreConfirmation
  W->>E: recordResult(confirmation)
  E-.->>U: SSE event(STORED)
```

## State machine — `MemoryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: MemoryEntrySanitized
  SANITIZED --> STORING: MemoryStoringStarted
  STORING --> STORED: MemoryEntryStored
  STORING --> FAILED: MemoryFailed (agent error)
  SUBMITTED --> FAILED: MemoryFailed (sanitizer error)
  STORED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  MemoryEntity ||--o{ MemoryEntrySubmitted : emits
  MemoryEntity ||--o{ MemoryEntrySanitized : emits
  MemoryEntity ||--o{ MemoryStoringStarted : emits
  MemoryEntity ||--o{ MemoryEntryStored : emits
  MemoryEntity ||--o{ MemoryFailed : emits
  MemoryView }o--|| MemoryEntity : projects
  MemorySanitizer }o--|| MemoryEntity : subscribes
  MemoryWorkflow }o--|| MemoryEntity : reads-and-writes
  MemoryAgent ||--o{ StoreConfirmation : returns
  MemoryAgent ||--o{ RecallResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MemoryEndpoint` | `api/MemoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MemoryEntity` | `application/MemoryEntity.java` (state in `domain/Memory.java`, events in `domain/MemoryEvent.java`) |
| `MemorySanitizer` | `application/MemorySanitizer.java` |
| `MemoryWorkflow` | `application/MemoryWorkflow.java` |
| `MemoryAgent` | `application/MemoryAgent.java` (tasks in `application/MemoryTasks.java`) |
| `MemoryView` | `application/MemoryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `storeStep` 30 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(MemoryWorkflow::error)`. The 30 s on `storeStep` accommodates LLM latency for both REMEMBER and RECALL task types (Lesson 4).
- **Idempotency**: every workflow uses `"memory-" + memoryId` as the workflow id; `MemorySanitizer` is allowed to redeliver `MemoryEntrySubmitted` events because `MemoryEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized entry is a no-op.
- **One agent, two task types**: `MemoryAgent` handles both `REMEMBER_CONTENT` and `RECALL_CONTENT` tasks. The instance id is `"memory-" + memoryId`, giving each operation its own conversation context. `maxIterationsPerTask(3)` caps retries per task.
- **Recall context injection**: for RECALL operations, `MemoryWorkflow.storeStep` fetches stored entries from `MemoryView` and injects them into the task instructions string before calling the agent. This keeps the single-agent invariant — no second agent mediates the context lookup.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
