# PLAN — akka-llm-client-basic

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ConversationEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  Agent[ConversationAgent]:::agent
  Guard[ReplyGuardrail]:::guard
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|createSession| Entity
  API -->|submitTurn| Entity
  API -->|runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|pass / reject| Agent
  Agent -->|ConversationReply| API
  API -->|recordReply| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ConversationEndpoint
  participant E as ConversationEntity
  participant A as ConversationAgent
  participant G as ReplyGuardrail

  U->>API: POST /api/conversations
  API->>E: createSession
  E-->>API: { sessionId }
  U->>API: POST /api/conversations/{sessionId}/turns
  API->>E: submitTurn(turnId, prompt)
  E-->>API: ok
  API->>A: runSingleTask(prompt + history)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>API: ConversationReply
  API->>E: recordReply(turnId, reply)
  E-.->>U: SSE event(REPLIED)
```

## State machine — `ConversationEntity` (session + turn)

```mermaid
stateDiagram-v2
  [*] --> ACTIVE: SessionCreated
  ACTIVE --> ACTIVE: TurnSubmitted
  ACTIVE --> ACTIVE: TurnReplied
  ACTIVE --> ACTIVE: TurnFailed
  ACTIVE --> CLOSED: closeSession
  state ACTIVE {
    [*] --> PENDING: TurnSubmitted
    PENDING --> REPLIED: TurnReplied
    PENDING --> FAILED: TurnFailed
  }
  CLOSED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ SessionCreated : emits
  ConversationEntity ||--o{ TurnSubmitted : emits
  ConversationEntity ||--o{ TurnReplied : emits
  ConversationEntity ||--o{ TurnFailed : emits
  ConversationView }o--|| ConversationEntity : projects
  ConversationAgent ||--o{ ConversationReply : returns
  ReplyGuardrail }o--|| ConversationAgent : guards
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversationEndpoint` | `api/ConversationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/ConversationSession.java`, events in `domain/ConversationEvent.java`) |
| `ConversationAgent` | `application/ConversationAgent.java` (tasks in `application/ConversationTasks.java`) |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Inline agent call**: the agent is invoked directly from `ConversationEndpoint`'s POST `/turns` handler via `componentClient.forAutonomousAgent(...).runSingleTask(...)`. No intermediate Workflow is needed — the handler blocks on the result with a 30 s server-side timeout. This is the minimal-path design for the baseline.
- **Per-session agent context**: the AutonomousAgent instance id is `"agent-" + sessionId`, giving each session its own conversation context. Prior turns are formatted and passed as instruction text on each call; the agent does not maintain its own internal turn log independently of the entity.
- **Guardrail-driven retry**: when `ReplyGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask(3)`; if all 3 iterations fail the guardrail, the endpoint's agent call throws, the handler calls `failTurn(reason)`, and the turn transitions to `FAILED`.
- **Idempotency**: `createSession` is the only minting operation; subsequent `submitTurn` calls on the same session append turns. The entity guards against duplicate `turnId` submissions.
- **No saga / no compensation**: the entity is append-only and every agent call is a single task. There is nothing external to roll back.
- **View query**: `ConversationView` exposes a single `getAllSessions` query; the UI filters and sorts client-side. No enum column indexing is attempted (Lesson 2).
