# PLAN — with Signals & Queries

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  API[ChatEndpoint]:::ep
  Entity[ChatSessionEntity]:::ese
  WF[ChatSessionWorkflow]:::wf
  Agent[ChatAgent]:::agent
  Validator[SignalValidator]:::guard
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|openSession| Entity
  API -->|start workflow| WF
  API -->|addTurnSignal| WF
  API -->|pauseSignal| WF
  API -->|resumeSignal| WF
  API -->|closeSignal| WF
  API -->|getSessionQuery| WF
  WF -->|initStep write| Entity
  WF -->|processTurnStep validate| Validator
  Validator -.->|before-agent-invocation| Agent
  Agent -->|AgentReply| WF
  WF -->|recordReply| Entity
  WF -->|pauseSession| Entity
  WF -->|resumeSession| Entity
  WF -->|closeSession| Entity
  WF -->|failSession| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, two turns)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ChatEndpoint
  participant E as ChatSessionEntity
  participant W as ChatSessionWorkflow
  participant V as SignalValidator
  participant A as ChatAgent

  U->>API: POST /api/sessions
  API->>E: openSession(sessionId, sessionName)
  API->>W: start(sessionId, initialPrompt)
  API-->>U: { sessionId }
  W->>E: initStep / SessionOpened
  W->>V: validate(AddTurnSignal{initialPrompt})
  V-->>W: accept
  W->>A: runSingleTask(context + prompt)
  A-->>W: AgentReply
  W->>E: recordReply(turnId, reply)
  E-.->>U: SSE event(ACTIVE, turn 1)

  U->>API: PUT /api/sessions/{id}/signal/turn
  API->>W: addTurnSignal(prompt2)
  W->>V: validate(AddTurnSignal{prompt2})
  V-->>W: accept
  W->>A: runSingleTask(context + prompt2)
  A-->>W: AgentReply
  W->>E: recordReply(turnId2, reply2)
  E-.->>U: SSE event(ACTIVE, turn 2)
```

## State machine — `ChatSessionEntity`

```mermaid
stateDiagram-v2
  [*] --> STARTING
  STARTING --> ACTIVE: SessionOpened
  ACTIVE --> ACTIVE: TurnAdded / TurnReplied
  ACTIVE --> PAUSED: SessionPaused
  PAUSED --> ACTIVE: SessionResumed
  ACTIVE --> CLOSING: closeSignal
  PAUSED --> CLOSING: closeSignal
  CLOSING --> CLOSED: SessionClosed
  ACTIVE --> FAILED: SessionFailed (guardrail or agent error)
  PAUSED --> FAILED: SessionFailed
  CLOSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ChatSessionEntity ||--o{ SessionOpened : emits
  ChatSessionEntity ||--o{ TurnAdded : emits
  ChatSessionEntity ||--o{ TurnReplied : emits
  ChatSessionEntity ||--o{ SessionPaused : emits
  ChatSessionEntity ||--o{ SessionResumed : emits
  ChatSessionEntity ||--o{ SessionClosed : emits
  ChatSessionEntity ||--o{ SessionFailed : emits
  SessionView }o--|| ChatSessionEntity : projects
  ChatSessionWorkflow }o--|| ChatSessionEntity : reads-and-writes
  ChatAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ChatSessionEntity` | `application/ChatSessionEntity.java` (state in `domain/SessionState.java`, events in `domain/SessionEvent.java`) |
| `ChatSessionWorkflow` | `application/ChatSessionWorkflow.java` |
| `ChatAgent` | `application/ChatAgent.java` (tasks in `application/ChatTasks.java`) |
| `SignalValidator` | `application/SignalValidator.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeouts**: `initStep` 5 s, `processTurnStep` 60 s, `pausedStep` 300 s, `closeStep` 5 s, `failStep` 5 s. Default step recovery `maxRetries(2).failoverTo(ChatSessionWorkflow::failStep)`. The 60 s on `processTurnStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: the workflow uses `"session-" + sessionId` as the workflow id; a duplicate `POST /api/sessions` with the same id is rejected at the entity level before the workflow starts.
- **One agent per session**: the AutonomousAgent instance id is `"chat-" + sessionId`. Each session has its own conversation context. `capability(...).maxIterationsPerTask(3)` caps retries.
- **Guardrail-driven rejection**: when `SignalValidator` rejects an inbound signal, the rejection propagates as a failed workflow step. The entity records `SessionFailed`; the UI displays the structured error message.
- **Query is side-effect-free**: `getSessionQuery()` reads the workflow's in-memory `SessionState` and returns it. It does not call `componentClient`, emit an event, or advance a step. The caller bears no latency beyond the workflow actor's mailbox processing time.
- **HITL pause window**: the `pausedStep` has a 300 s timeout. If no `resumeSignal` or `closeSignal` arrives within 5 minutes, the step times out with the configured recovery; the entity transitions to `FAILED`. Deployers extending this blueprint for production should increase the timeout or implement a durable pause pattern.
- **No saga / no compensation**: every step is either a pure read, an append-only entity write, or a single-task agent call with a bounded iteration budget.
