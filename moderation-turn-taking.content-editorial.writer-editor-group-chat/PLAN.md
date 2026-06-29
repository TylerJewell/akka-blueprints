# PLAN — writer-editor-group-chat

Architectural sketch. All four mermaid diagrams plus the component table. Diagrams render on the Architecture tab; they inherit the Lesson 24 state-label CSS overrides and theme variables.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  ChatEndpoint:::ep -->|enqueueTopic| InboundTopicQueue:::ese
  TopicSimulator:::ta -.->|drip topic| InboundTopicQueue
  InboundTopicQueue -.->|TopicQueued| ConversationConsumer:::cons
  ConversationConsumer -->|start| GroupChatWorkflow:::wf
  GroupChatWorkflow -->|grant + runSingleTask| WriterAgent:::agent
  GroupChatWorkflow -->|grant + runSingleTask| EditorAgent:::agent
  GroupChatWorkflow -->|publishTurn / blockTurn / approve| ConversationEntity:::ese
  ConversationEntity -.->|events| TurnView:::view
  ConversationEntity -.->|TurnPublished EDITOR| TurnEvalConsumer:::cons
  TurnEvalConsumer -->|record score| ConversationEntity
  StallMonitor:::ta -.->|escalate stalled| ConversationEntity
  TurnView -->|query| ChatEndpoint
  AppEndpoint:::ep -->|serve UI| TurnView
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant EP as ChatEndpoint
  participant Q as InboundTopicQueue
  participant C as ConversationConsumer
  participant W as GroupChatWorkflow
  participant WA as WriterAgent
  participant EA as EditorAgent
  participant E as ConversationEntity
  U->>EP: POST /api/conversations { topic }
  EP->>Q: enqueueTopic(topic)
  Q-->>C: TopicQueued
  C->>W: start(conversationId, topic)
  W->>E: open
  loop until approved or maxRounds
    Note over W: grant floor to Writer (RequestToSpeak)
    W->>WA: runSingleTask(WRITE)
    WA-->>W: Draft
    Note over W: before-agent-response guardrail
    W->>E: publishTurn(WRITER) or blockTurn
    Note over W: grant floor to Editor
    W->>EA: runSingleTask(CRITIQUE)
    EA-->>W: Critique
    W->>E: publishTurn(EDITOR)
  end
  W->>E: approve | markMaxRounds
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> IN_PROGRESS: ConversationOpened
  IN_PROGRESS --> IN_PROGRESS: TurnPublished / TurnBlocked
  IN_PROGRESS --> APPROVED: ConversationApproved
  IN_PROGRESS --> MAX_ROUNDS_REACHED: MaxRoundsReached
  IN_PROGRESS --> ESCALATED: ConversationEscalated
  APPROVED --> CLOSED: ConversationClosed
  MAX_ROUNDS_REACHED --> CLOSED: ConversationClosed
  ESCALATED --> CLOSED: ConversationClosed
  CLOSED --> [*]
```

## Entity model

```mermaid
erDiagram
  CONVERSATION ||--o{ TURN : contains
  CONVERSATION {
    string id
    string topic
    string status
    string currentSpeaker
    int round
  }
  TURN {
    string speaker
    string content
    int round
    double evalScore
    boolean blocked
  }
  CONVERSATION ||--o{ CONVERSATION_EVENT : emits
  CONVERSATION_EVENT {
    string type
    instant at
  }
  CONVERSATION ||--|| TURN_VIEW_ROW : projects
```

## Component table

| Component | Akka primitive | Path (generated) |
|---|---|---|
| WriterAgent | AutonomousAgent | `application/WriterAgent.java` |
| EditorAgent | AutonomousAgent | `application/EditorAgent.java` |
| GroupChatTasks | task constants | `application/GroupChatTasks.java` |
| GroupChatWorkflow | Workflow | `application/GroupChatWorkflow.java` |
| ConversationEntity | EventSourcedEntity | `domain/ConversationEntity.java` |
| InboundTopicQueue | EventSourcedEntity | `domain/InboundTopicQueue.java` |
| TurnView | View | `application/TurnView.java` |
| ConversationConsumer | Consumer | `application/ConversationConsumer.java` |
| TurnEvalConsumer | Consumer | `application/TurnEvalConsumer.java` |
| TopicSimulator | TimedAction | `application/TopicSimulator.java` |
| StallMonitor | TimedAction | `application/StallMonitor.java` |
| ChatEndpoint | HttpEndpoint | `api/ChatEndpoint.java` |
| AppEndpoint | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts.** `writerTurnStep` and `editorTurnStep` call agents; each overrides `stepTimeout` to 60 s (Lesson 4). Default 5 s would time out every LLM call. `defaultStepRecovery(maxRetries(2).failoverTo(error))`.
- **Idempotency.** Each workflow instance is keyed by the conversation UUID minted by `ConversationConsumer`; re-delivery of `TopicQueued` does not start a second workflow for the same queued offset.
- **Round-robin bound.** The manager loops Writer→Editor up to `maxRounds` (default 4). A blocked turn re-requests the same speaker with bounded retries before failing the step over to `error`.
- **No saga.** All actions are in-process and event-sourced; there is no external side effect to compensate. Escalation is a forward transition driven by `StallMonitor`, not a rollback.
