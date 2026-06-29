# PLAN — restaurant-assistant

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
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[SessionEndpoint]:::ep
  Entity[SessionEntity]:::ese
  WF[SessionWorkflow]:::wf
  Agent[RestaurantAssistantAgent]:::agent
  RG[ResponseGuardrail]:::guard
  TCG[ToolCallGuardrail]:::guard
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|open / addMessage| Entity
  API -->|start / resume| WF
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-agent-response| RG
  Agent -.->|before-tool-call| TCG
  Agent -->|AssistantResponse| WF
  WF -->|recordAgentReply| Entity
  WF -->|confirmReservation / commitOrder| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J2 (reservation path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Customer (UI)
  participant API as SessionEndpoint
  participant E as SessionEntity
  participant W as SessionWorkflow
  participant A as RestaurantAssistantAgent
  participant RG as ResponseGuardrail
  participant TCG as ToolCallGuardrail

  U->>API: POST /api/sessions/{id}/messages
  API->>E: addMessage(text)
  API->>W: resume(sessionId, text)
  W->>A: runSingleTask(sessionContext + message)
  A->>RG: before-agent-response(candidate)
  RG-->>A: accept
  A->>TCG: before-tool-call(makeReservation, args)
  TCG-->>A: accept
  A-->>W: AssistantResponse(toolName=makeReservation)
  W->>E: confirmReservation(reservation)
  E-->>W: ReservationConfirmed
  W->>E: recordAgentReply(confirmationText)
  E-.->>U: SSE event(RESERVATION_HELD)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> ACTIVE: MessageAdded (first customer message)
  ACTIVE --> ACTIVE: AgentReplied (no tool call)
  ACTIVE --> RESERVATION_HELD: ReservationConfirmed
  ACTIVE --> ORDER_PLACED: OrderCommitted
  RESERVATION_HELD --> ACTIVE: AgentReplied (follow-up)
  RESERVATION_HELD --> ORDER_PLACED: OrderCommitted
  ORDER_PLACED --> CLOSED: SessionClosed
  RESERVATION_HELD --> CLOSED: SessionClosed
  ACTIVE --> CLOSED: SessionClosed
  ACTIVE --> FAILED: SessionFailed (agent error)
  CLOSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionOpened : emits
  SessionEntity ||--o{ MessageAdded : emits
  SessionEntity ||--o{ AgentReplied : emits
  SessionEntity ||--o{ ReservationConfirmed : emits
  SessionEntity ||--o{ OrderCommitted : emits
  SessionEntity ||--o{ SessionClosed : emits
  SessionEntity ||--o{ SessionFailed : emits
  SessionView }o--|| SessionEntity : projects
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  RestaurantAssistantAgent ||--o{ AssistantResponse : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `RestaurantAssistantAgent` | `application/RestaurantAssistantAgent.java` (tasks in `application/SessionTasks.java`) |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `MenuCatalog` | `application/MenuCatalog.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `agentStep` 60 s, `commitStep` 10 s, `closeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SessionWorkflow::error)`. The 60 s on `agentStep` accommodates LLM latency including guardrail retries (Lesson 4).
- **Idempotency**: every workflow uses `"session-" + sessionId` as the workflow id. A second `POST /sessions/{id}/messages` call resumes the existing workflow rather than starting a new one. `SessionEntity.addMessage` is event-version-guarded to prevent duplicate message events.
- **One agent per session**: the AutonomousAgent instance id is `"assistant-" + sessionId`, giving each session its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Dual-guardrail order**: `ResponseGuardrail` fires on the complete candidate reply; `ToolCallGuardrail` fires on the extracted tool-call arguments before the workflow receives them. They are independent checks at different cut points.
- **No saga / no compensation**: reservations and orders are append-only events on the entity. There is no external booking system to roll back in this baseline.
