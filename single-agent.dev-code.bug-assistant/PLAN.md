# PLAN — bug-assistant

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

  API[BugEndpoint]:::ep
  Entity[BugEntity]:::ese
  Consumer[TicketSyncConsumer]:::cons
  WF[BugWorkflow]:::wf
  Agent[BugResolutionAgent]:::agent
  Guard[TicketWriteGuardrail]:::guard
  View[BugView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|BugSubmitted| Consumer
  Consumer -->|attachEnriched| Entity
  Consumer -->|start workflow| WF
  WF -->|awaitEnrichedStep poll| Entity
  WF -->|investigateStep runSingleTask| Agent
  Agent -.->|before-tool-call write_ticket| Guard
  Guard -.->|pass or reject| Agent
  Agent -->|Resolution| WF
  WF -->|recordResolution| Entity
  WF -->|closeStep close| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as BugEndpoint
  participant E as BugEntity
  participant C as TicketSyncConsumer
  participant W as BugWorkflow
  participant A as BugResolutionAgent
  participant G as TicketWriteGuardrail

  U->>API: POST /api/bugs
  API->>E: submit(report)
  E-->>API: { bugId }
  E-.->>C: BugSubmitted
  C->>C: build TicketMetadata
  C->>E: attachEnriched(ticket)
  C->>W: start(bugId)
  W->>E: poll getBug
  E-->>W: ticket.isPresent()
  W->>E: markInvestigating
  W->>A: runSingleTask(bugDetails)
  A->>A: search_web(query)
  A->>A: read_ticket(ticketId)
  A->>G: before-tool-call write_ticket(...)
  G-->>A: accept
  A-->>W: Resolution
  W->>E: recordResolution(resolution)
  W->>E: close()
  E-.->>U: SSE event(RESOLVED)
```

## State machine — `BugEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> ENRICHED: TicketEnriched
  ENRICHED --> INVESTIGATING: InvestigationStarted
  INVESTIGATING --> RESOLVED: ResolutionRecorded
  SUBMITTED --> FAILED: BugFailed (enrichment error)
  INVESTIGATING --> FAILED: BugFailed (agent error / guardrail exhausted)
  RESOLVED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BugEntity ||--o{ BugSubmitted : emits
  BugEntity ||--o{ TicketEnriched : emits
  BugEntity ||--o{ InvestigationStarted : emits
  BugEntity ||--o{ ResolutionRecorded : emits
  BugEntity ||--o{ BugFailed : emits
  BugView }o--|| BugEntity : projects
  TicketSyncConsumer }o--|| BugEntity : subscribes
  BugWorkflow }o--|| BugEntity : reads-and-writes
  BugResolutionAgent ||--o{ Resolution : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BugEndpoint` | `api/BugEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BugEntity` | `application/BugEntity.java` (state in `domain/Bug.java`, events in `domain/BugEvent.java`) |
| `TicketSyncConsumer` | `application/TicketSyncConsumer.java` |
| `BugWorkflow` | `application/BugWorkflow.java` |
| `BugResolutionAgent` | `application/BugResolutionAgent.java` (tasks in `application/BugTasks.java`) |
| `TicketWriteGuardrail` | `application/TicketWriteGuardrail.java` |
| `BugView` | `application/BugView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitEnrichedStep` 15 s, `investigateStep` 90 s, `closeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BugWorkflow::error)`. The 90 s on `investigateStep` accommodates multi-turn tool-call sequences (Lesson 4).
- **Idempotency**: every workflow uses `"bug-" + bugId` as the workflow id; the `TicketSyncConsumer` Consumer is allowed to redeliver `BugSubmitted` events because `BugEntity.attachEnriched` is event-version-guarded — a second enrichment attempt against an already-enriched bug is a no-op.
- **One agent per bug**: the AutonomousAgent instance id is `"resolver-" + bugId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(5)` caps guardrail-triggered retries at 5.
- **Guardrail-driven retry**: when `TicketWriteGuardrail` rejects a `write_ticket` call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 5 iterations fail validation, the workflow's `investigateStep` fails over to `error` and the entity transitions to `FAILED`.
- **Tool calls are simulated in-process**: `search_web` and `read_ticket` are backed by the mock-search-results.jsonl seed data. No external HTTP call happens; this keeps the blueprint self-contained.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
