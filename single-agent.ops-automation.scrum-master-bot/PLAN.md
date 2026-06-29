# PLAN — scrum-master-bot

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
  classDef svc fill:#1a1a1a,stroke:#888,color:#888;

  API[StandupEndpoint]:::ep
  Entity[StandupEntity]:::ese
  Consumer[SprintConsumer]:::cons
  WF[StandupWorkflow]:::wf
  Agent[ScrumMasterAgent]:::agent
  Guard[TicketWriteGuardrail]:::guard
  Poster[TicketPostingService]:::svc
  View[StandupView]:::view
  App[AppEndpoint]:::ep

  API -->|activate| Entity
  Entity -.->|SprintActivated| Consumer
  Consumer -->|attachSprintContext| Entity
  Consumer -->|start workflow| WF
  WF -->|collectSprintStep poll| Entity
  WF -->|standupStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|postTicketUpdate allowed| Poster
  Agent -->|StandupSummary| WF
  WF -->|recordSummary| Entity
  WF -->|postStep| Poster
  Poster -->|PostResult| WF
  WF -->|recordPostResult| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as StandupEndpoint
  participant E as StandupEntity
  participant C as SprintConsumer
  participant W as StandupWorkflow
  participant A as ScrumMasterAgent
  participant G as TicketWriteGuardrail
  participant P as TicketPostingService

  U->>API: POST /api/standups
  API->>E: activate(sprintId, members, tickets)
  E-->>API: { sessionId }
  E-.->>C: SprintActivated
  C->>C: build SprintContext
  C->>E: attachSprintContext
  C->>W: start(sessionId)
  W->>E: poll getSession
  E-->>W: sprintContext.isPresent()
  W->>E: markRunning
  W->>A: runSingleTask(sprint context)
  A->>G: before-tool-call(postTicketUpdate, ticketId)
  G-->>A: allowed (ticket in scope)
  A->>P: postTicketUpdate(ticketId, comment)
  P-->>A: posted
  A-->>W: StandupSummary
  W->>E: recordSummary(summary)
  W->>P: post remaining tickets
  P-->>W: PostResult
  W->>E: recordPostResult(result)
  E-.->>U: SSE event(POSTED)
```

## State machine — `StandupEntity`

```mermaid
stateDiagram-v2
  [*] --> COLLECTING
  COLLECTING --> COLLECTING: SprintContextAttached
  COLLECTING --> RUNNING: StandupStarted
  RUNNING --> SUMMARY_READY: SummaryRecorded
  SUMMARY_READY --> POSTED: UpdatesPosted
  RUNNING --> FAILED: SessionFailed (agent error)
  COLLECTING --> FAILED: SessionFailed (context error)
  POSTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  StandupEntity ||--o{ SprintActivated : emits
  StandupEntity ||--o{ SprintContextAttached : emits
  StandupEntity ||--o{ StandupStarted : emits
  StandupEntity ||--o{ SummaryRecorded : emits
  StandupEntity ||--o{ UpdatesPosted : emits
  StandupEntity ||--o{ SessionFailed : emits
  StandupView }o--|| StandupEntity : projects
  SprintConsumer }o--|| StandupEntity : subscribes
  StandupWorkflow }o--|| StandupEntity : reads-and-writes
  ScrumMasterAgent ||--o{ StandupSummary : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `StandupEndpoint` | `api/StandupEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `StandupEntity` | `application/StandupEntity.java` (state in `domain/StandupSession.java`, events in `domain/SessionEvent.java`) |
| `SprintConsumer` | `application/SprintConsumer.java` |
| `StandupWorkflow` | `application/StandupWorkflow.java` |
| `ScrumMasterAgent` | `application/ScrumMasterAgent.java` (tasks in `application/StandupTasks.java`) |
| `TicketWriteGuardrail` | `application/TicketWriteGuardrail.java` |
| `TicketPostingService` | `application/TicketPostingService.java` |
| `StandupView` | `application/StandupView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `collectSprintStep` 15 s, `standupStep` 120 s, `postStep` 30 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(StandupWorkflow::error)`. The 120 s on `standupStep` accommodates LLM latency and multiple tool-call iterations (Lesson 4).
- **Idempotency**: every workflow uses `"standup-" + sessionId` as the workflow id; the `SprintConsumer` Consumer is allowed to redeliver `SprintActivated` because `StandupEntity.attachSprintContext` is event-version-guarded — a second attach against an already-context-bearing session is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"scrum-" + sessionId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered write attempts at 4.
- **Guardrail-driven skip**: when `TicketWriteGuardrail` rejects a `postTicketUpdate` call, the rejection is returned as a structured error to the agent loop. The agent counts it as a used iteration and must either select a valid ticket or produce its summary without that update.
- **Post step is synchronous and deterministic**: `TicketPostingService` runs in-process inside `postStep`. No LLM call — it is a plain service class. The same inputs always produce the same (simulated) output.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
