# PLAN — bidi-demo

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

  API[ChannelEndpoint]:::ep
  Entity[ChannelEntity]:::ese
  Forwarder[MessageForwarder]:::cons
  WF[ChannelWorkflow]:::wf
  Agent[ChannelAgent]:::agent
  Guard[FrameGuardrail]:::guard
  View[ChannelView]:::view
  App[AppEndpoint]:::ep

  API -->|open / receiveMessage| Entity
  Entity -.->|MessageReceived| Forwarder
  Forwarder -->|startTurn| Entity
  Forwarder -->|start workflow| WF
  WF -->|awaitTurnStep poll| Entity
  WF -->|respondStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|ResponseFrameList| WF
  WF -->|publishFrame x N| Entity
  WF -->|completeTurn| Entity
  WF -->|flushStep closeChannel?| Entity
  Entity -.->|projects| View
  API -->|list / SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ChannelEndpoint
  participant E as ChannelEntity
  participant F as MessageForwarder
  participant W as ChannelWorkflow
  participant A as ChannelAgent
  participant G as FrameGuardrail

  U->>API: POST /api/channels
  API->>E: open(channelName, budget)
  E-->>API: { channelId }
  U->>API: POST /api/channels/{id}/messages
  API->>E: receiveMessage(content, sentBy)
  E-->>API: { messageId }
  E-.->>F: MessageReceived
  F->>E: startTurn(turnId, messageId)
  F->>W: start(channelId, turnId)
  W->>E: poll getChannel
  E-->>W: turn.status == PROCESSING
  W->>E: TurnStarted emitted
  W->>A: runSingleTask(message + history attachment)
  A->>G: before-agent-response(candidate frames)
  G-->>A: accept
  A-->>W: ResponseFrameList
  loop for each frame
    W->>E: publishFrame(frame)
    E-.->>U: SSE frame event
  end
  W->>E: completeTurn(turnId)
  W->>E: flushStep — budget check
  E-.->>U: SSE TurnCompleted event
```

## State machine — `ChannelEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> ACTIVE: MessageReceived (first message)
  ACTIVE --> ACTIVE: MessageReceived (subsequent)
  ACTIVE --> CLOSED: ChannelClosed (budget exhausted)
  OPEN --> FAILED: ChannelClosed (error)
  ACTIVE --> FAILED: ChannelClosed (error)
  CLOSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ChannelEntity ||--o{ ChannelOpened : emits
  ChannelEntity ||--o{ MessageReceived : emits
  ChannelEntity ||--o{ TurnStarted : emits
  ChannelEntity ||--o{ FramePublished : emits
  ChannelEntity ||--o{ TurnCompleted : emits
  ChannelEntity ||--o{ TurnFailed : emits
  ChannelEntity ||--o{ ChannelClosed : emits
  ChannelView }o--|| ChannelEntity : projects
  MessageForwarder }o--|| ChannelEntity : subscribes
  ChannelWorkflow }o--|| ChannelEntity : reads-and-writes
  ChannelAgent ||--o{ ResponseFrameList : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChannelEndpoint` | `api/ChannelEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ChannelEntity` | `application/ChannelEntity.java` (state in `domain/Channel.java`, events in `domain/ChannelEvent.java`) |
| `MessageForwarder` | `application/MessageForwarder.java` |
| `ChannelWorkflow` | `application/ChannelWorkflow.java` |
| `ChannelAgent` | `application/ChannelAgent.java` (tasks in `application/ChannelTasks.java`) |
| `FrameGuardrail` | `application/FrameGuardrail.java` |
| `ChannelView` | `application/ChannelView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitTurnStep` 10 s, `respondStep` 60 s, `flushStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ChannelWorkflow::error)`. The 60 s on `respondStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"workflow-" + turnId` as its id; `MessageForwarder` may redeliver `MessageReceived` events, but `ChannelEntity.startTurn` is event-version-guarded — a duplicate startTurn against an already-started turn is a no-op.
- **One agent per channel**: the `ChannelAgent` instance id is `"agent-" + channelId`, so the same conversation context accumulates across turns on that channel. `maxIterationsPerTask(3)` caps guardrail-triggered retries.
- **Frame-by-frame publishing**: `respondStep` iterates the returned `ResponseFrameList` and calls `ChannelEntity.publishFrame` for each. Each `FramePublished` event propagates to the SSE stream immediately, giving the client a streaming experience without requiring the agent to stream at the protocol level.
- **Turn budget enforcement**: `flushStep` checks `channel.turns().size() >= channel.turnBudget()` and calls `closeChannel` if true. The check is synchronous and deterministic — no LLM call.
- **No saga / no compensation**: frame publishing is append-only. A failed `respondStep` calls `failTurn`; partial frames remain in the entity log for audit but the turn is marked `FAILED`.
