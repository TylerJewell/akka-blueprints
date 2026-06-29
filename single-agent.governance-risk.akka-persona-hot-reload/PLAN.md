# PLAN — persona-hot-reload

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
  classDef gate fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef eval fill:#0e1a2a,stroke:#58a6ff,color:#58a6ff;

  API[PersonaEndpoint]:::ep
  Entity[PersonaEntity]:::ese
  Consumer[PersonaChangeConsumer]:::cons
  Validator[PersonaValidator]:::gate
  WF[PersonaWatchWorkflow]:::wf
  Agent[PersonaAgent]:::agent
  Revalidator[BehavioralRevalidator]:::eval
  View[PersonaView]:::view
  App[AppEndpoint]:::ep

  API -->|propose| Entity
  Entity -.->|PersonaChangeProposed| Consumer
  Consumer -->|validate| Validator
  Validator -->|ValidationResult| Consumer
  Consumer -->|activate or reject| Entity
  Consumer -->|start workflow| WF
  WF -->|awaitValidationStep poll| Entity
  WF -->|activateAgentStep rebuild| Agent
  WF -->|revalidationStep probes| Agent
  Agent -->|ProbeResult| Revalidator
  Revalidator -->|RevalidationResult| WF
  WF -->|recordRevalidation| Entity
  WF -->|monitoringStep window| Consumer
  Consumer -->|closeMonitoringWindow| Entity
  Entity -.->|projects| View
  API -->|list/SSE/chat| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant Op as Operator (UI)
  participant API as PersonaEndpoint
  participant E as PersonaEntity
  participant C as PersonaChangeConsumer
  participant V as PersonaValidator
  participant W as PersonaWatchWorkflow
  participant A as PersonaAgent
  participant R as BehavioralRevalidator

  Op->>API: POST /api/personas/push
  API->>E: propose(snapshot)
  E-->>API: { changeId }
  E-.->>C: PersonaChangeProposed
  C->>V: validate(snapshot)
  V-->>C: ValidationResult{valid=true}
  C->>E: activate(changeId)
  C->>W: start(changeId)
  W->>E: poll getChange
  E-->>W: status==ACTIVATING
  W->>A: rebuildDefinition(snapshot)
  W->>E: markActive(changeId)
  W->>A: probe #1..N
  A-->>R: probe responses
  R-->>W: RevalidationResult{PASS}
  W->>E: recordRevalidation(result)
  W->>C: open monitoring window (300s)
  C->>E: openMonitoringWindow
  Note over C,E: 300s forward window
  C->>E: closeMonitoringWindow(count)
  E-.->>Op: SSE event(MONITORED)
```

## State machine — `PersonaEntity`

```mermaid
stateDiagram-v2
  [*] --> VALIDATING
  VALIDATING --> ACTIVATING: PersonaActivated
  VALIDATING --> REJECTED: PersonaRejected (gate blocked)
  ACTIVATING --> ACTIVE: PersonaAgent rebuilt
  ACTIVE --> REVALIDATING: revalidationStep started
  REVALIDATING --> MONITORED: PersonaRevalidated + window closed
  REVALIDATING --> FAILED: PersonaChangeFailed (revalidation error)
  ACTIVATING --> FAILED: PersonaChangeFailed (rebuild error)
  MONITORED --> [*]
  REJECTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PersonaEntity ||--o{ PersonaChangeProposed : emits
  PersonaEntity ||--o{ PersonaActivated : emits
  PersonaEntity ||--o{ PersonaRejected : emits
  PersonaEntity ||--o{ PersonaRevalidated : emits
  PersonaEntity ||--o{ MonitoringWindowOpened : emits
  PersonaEntity ||--o{ MonitoringWindowClosed : emits
  PersonaEntity ||--o{ PersonaChangeFailed : emits
  PersonaView }o--|| PersonaEntity : projects
  PersonaChangeConsumer }o--|| PersonaEntity : subscribes
  PersonaWatchWorkflow }o--|| PersonaEntity : reads-and-writes
  PersonaAgent ||--o{ AgentResponse : returns
  BehavioralRevalidator ||--o{ RevalidationResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PersonaEndpoint` | `api/PersonaEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PersonaEntity` | `application/PersonaEntity.java` (state in `domain/PersonaChange.java`, events in `domain/PersonaEvent.java`) |
| `PersonaChangeConsumer` | `application/PersonaChangeConsumer.java` |
| `PersonaWatchWorkflow` | `application/PersonaWatchWorkflow.java` |
| `PersonaAgent` | `application/PersonaAgent.java` (tasks in `application/PersonaTasks.java`) |
| `PersonaValidator` | `application/PersonaValidator.java` |
| `BehavioralRevalidator` | `application/BehavioralRevalidator.java` |
| `PersonaView` | `application/PersonaView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitValidationStep` 10 s, `activateAgentStep` 10 s, `revalidationStep` 60 s, `monitoringStep` 320 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(PersonaWatchWorkflow::error)`. The 60 s on `revalidationStep` accommodates back-to-back probe LLM calls (Lesson 4).
- **Idempotency**: every workflow uses `"persona-" + changeId` as the workflow id; `PersonaChangeConsumer` is allowed to redeliver `PersonaChangeProposed` events because `PersonaEntity.activate` is version-guarded — a duplicate activate for an already-active change is a no-op.
- **One agent per deployment**: `PersonaAgent` uses a fixed instance id `"persona-agent"`. All chat queries and all probe calls share that instance; the definition is rebuilt in-place via `rebuildDefinition`. The `maxIterationsPerTask(2)` cap prevents runaway probe loops.
- **Gate is synchronous**: `PersonaValidator.validate` runs inside `PersonaChangeConsumer` before any activation event is written. If the validator throws, the Consumer treats it as a rejection.
- **Revalidation is deterministic**: `BehavioralRevalidator` scores ProbeResults in-process. No LLM call — the same probe responses always yield the same RevalidationStatus. This is a deliberate single-agent guarantee.
- **Monitoring window is best-effort**: if `PersonaChangeConsumer` crashes during the window, forwarded-count may undercount. The window still closes at `closedAt`; the entity record shows the partial count. No saga compensation is needed — the window is observational, not transactional.
