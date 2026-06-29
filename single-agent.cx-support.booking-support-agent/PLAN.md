# PLAN — booking-support-agent

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
  classDef store fill:#0f1f2a,stroke:#60a5fa,color:#60a5fa;

  API[SupportEndpoint]:::ep
  Entity[BookingSessionEntity]:::ese
  Sanitizer[PiiSanitizer]:::cons
  WF[BookingSessionWorkflow]:::wf
  Agent[BookingSupportAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  Store[BookingStore]:::store
  View[BookingSessionView]:::view
  App[AppEndpoint]:::ep

  API -->|createSession / receiveMessage| Entity
  Entity -.->|CustomerMessageReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|agentTurnStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|lookUpBooking| Store
  Agent -->|AgentReply| WF
  WF -->|recordAgentTurn| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, booking lookup)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SupportEndpoint
  participant E as BookingSessionEntity
  participant S as PiiSanitizer
  participant W as BookingSessionWorkflow
  participant A as BookingSupportAgent
  participant G as ToolCallGuardrail
  participant Bs as BookingStore

  U->>API: POST /api/sessions/{id}/turns
  API->>E: receiveMessage(customerMessage)
  E-->>API: { turnId }
  E-.->>S: CustomerMessageReceived
  S->>S: redact PII
  S->>E: attachSanitized(turnId, sanitized)
  S->>W: start(sessionId, turnId)
  W->>E: poll getSession
  E-->>W: turn.sanitized.isPresent()
  W->>A: runSingleTask(sanitizedMessage + history)
  A->>G: before-tool-call(lookUpBooking, BK-001)
  G->>Bs: lookUpBooking(BK-001)
  Bs-->>G: BookingRecord
  G-->>A: ALLOW
  A->>Bs: lookUpBooking(BK-001)
  Bs-->>A: BookingRecord
  A-->>W: AgentReply (replyText + toolCalls)
  W->>E: recordAgentTurn(turnId, reply)
  E-.->>U: SSE event(AGENT_REPLIED)
```

## State machine — `BookingSessionEntity` (turn lifecycle)

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: MessageSanitized
  SANITIZED --> AGENT_REPLIED: AgentTurnCompleted
  RECEIVED --> FAILED: TurnFailed (sanitizer error)
  SANITIZED --> FAILED: TurnFailed (agent error)
  AGENT_REPLIED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BookingSessionEntity ||--o{ SessionCreated : emits
  BookingSessionEntity ||--o{ CustomerMessageReceived : emits
  BookingSessionEntity ||--o{ MessageSanitized : emits
  BookingSessionEntity ||--o{ AgentTurnCompleted : emits
  BookingSessionEntity ||--o{ TurnFailed : emits
  BookingSessionView }o--|| BookingSessionEntity : projects
  PiiSanitizer }o--|| BookingSessionEntity : subscribes
  BookingSessionWorkflow }o--|| BookingSessionEntity : reads-and-writes
  BookingSupportAgent ||--o{ AgentReply : returns
  BookingStore ||--o{ BookingRecord : serves
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SupportEndpoint` | `api/SupportEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BookingSessionEntity` | `application/BookingSessionEntity.java` (state in `domain/BookingSession.java`, events in `domain/BookingSessionEvent.java`) |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `BookingSessionWorkflow` | `application/BookingSessionWorkflow.java` |
| `BookingSupportAgent` | `application/BookingSupportAgent.java` (tasks in `application/BookingSupportTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `BookingStore` | `application/BookingStore.java` |
| `BookingSessionView` | `application/BookingSessionView.java` |
| `JudgeAssertions` | `test/JudgeAssertions.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `agentTurnStep` 60 s, `recordTurnStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BookingSessionWorkflow::error)`. The 60 s on `agentTurnStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"turn-" + turnId` as the workflow id; `PiiSanitizer` Consumer is allowed to redeliver `CustomerMessageReceived` events because `BookingSessionEntity.attachSanitized` is turn-version-guarded — a second sanitize attempt for an already-sanitized turn is a no-op.
- **One agent per session**: `BookingSupportAgent` instance id is `"support-" + sessionId`, giving each session its own conversation context across turns. `capability(...).maxIterationsPerTask(4)` allows up to 4 tool-call iterations per turn, enough for a lookup → modify → confirm flow.
- **Guardrail and tool execution**: `ToolCallGuardrail` runs as `before-tool-call`; a BLOCK prevents the tool from executing. `BookingStore` serves only read-only lookups in the baseline — all mutations are recorded as entity events rather than written back to the store.
- **CI gate timing**: `JudgeAssertions` runs during `mvn test` only (not at runtime). It requires a model provider key or mock mode. The CI gate is a build-time concern; it does not add any runtime latency.
- **No saga / no compensation**: workflow steps are either read-only, append-only entity writes, or single-task agent calls. There is nothing external to roll back. Booking mutations in a real deployment would need compensation logic outside this baseline's scope.
