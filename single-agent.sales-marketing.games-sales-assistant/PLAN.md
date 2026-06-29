# PLAN — games-sales-assistant

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
  classDef idx fill:#0e1e20,stroke:#22d3ee,color:#22d3ee;

  API[SessionEndpoint]:::ep
  Entity[SessionEntity]:::ese
  Catalog[CatalogConsumer]:::cons
  Index[CatalogIndex]:::idx
  WF[SessionWorkflow]:::wf
  Agent[SalesAssistantAgent]:::agent
  Guard[ResponseGuardrail]:::guard
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|createSession| Entity
  API -->|postTurn| WF
  Entity -.->|projects| View
  Catalog -->|upsert| Index
  WF -->|initStep| Entity
  WF -->|assistStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|CatalogIndex lookup| Index
  Agent -->|AssistantResponse| WF
  WF -->|recordTurnAnswered| Entity
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SessionEndpoint
  participant E as SessionEntity
  participant W as SessionWorkflow
  participant A as SalesAssistantAgent
  participant G as ResponseGuardrail
  participant I as CatalogIndex

  U->>API: POST /api/sessions
  API->>E: createSession(shopperId)
  E-->>API: { sessionId }
  W->>E: initStep emit SessionCreated
  U->>API: POST /api/sessions/{id}/turns
  API->>W: assistStep(turnRequest)
  W->>E: emit TurnStarted
  W->>A: runSingleTask(question + context.json)
  A->>G: before-agent-response(candidate)
  G->>I: titleId lookup
  I-->>G: all titles valid
  G-->>A: accept
  A-->>W: AssistantResponse
  W->>E: recordTurnAnswered(turnId, response)
  E-.->>U: SSE event(TURN_ANSWERED)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> ACTIVE: TurnStarted (first turn)
  ACTIVE --> ACTIVE: TurnAnswered / TurnRejected / TurnFailed
  ACTIVE --> CLOSED: SessionClosed
  CREATED --> FAILED: SessionFailed (init error)
  ACTIVE --> FAILED: SessionFailed (unrecoverable agent error)
  CLOSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ TurnStarted : emits
  SessionEntity ||--o{ TurnAnswered : emits
  SessionEntity ||--o{ TurnRejected : emits
  SessionEntity ||--o{ TurnFailed : emits
  SessionEntity ||--o{ SessionClosed : emits
  SessionEntity ||--o{ SessionFailed : emits
  SessionView }o--|| SessionEntity : projects
  CatalogConsumer }o--|| CatalogIndex : hydrates
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  SalesAssistantAgent ||--o{ AssistantResponse : returns
  ResponseGuardrail }o--|| CatalogIndex : validates-against
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `CatalogConsumer` | `application/CatalogConsumer.java` |
| `CatalogIndex` | `application/CatalogIndex.java` |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `SalesAssistantAgent` | `application/SalesAssistantAgent.java` (tasks in `application/SalesTasks.java`) |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `initStep` 5 s, `assistStep` 45 s, `closeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SessionWorkflow::error)`. The 45 s on `assistStep` accommodates LLM latency (Lesson 4).
- **Session-scoped agent**: the AutonomousAgent instance id is `"assistant-" + sessionId`, which gives each session its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ResponseGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, `assistStep` fails over to `error` and the entity records `TurnRejected` with the final rejection code.
- **Idempotency**: workflow id is `"session-" + sessionId`; each turn within the session is handled as a distinct `assistStep` call with `turnId` as the task differentiator.
- **CatalogIndex hot-reload**: `CatalogConsumer` may receive multiple `CatalogUpdated` events (e.g., bulk catalog refresh). `CatalogIndex.upsert` is thread-safe; partial updates are safe because the index is only used for title-existence validation, not for price computation.
- **No saga / no compensation**: every step is either a pure write to the entity or a single-task agent call. There is nothing external to roll back.
