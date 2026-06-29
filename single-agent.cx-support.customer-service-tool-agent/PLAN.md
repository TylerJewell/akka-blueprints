# PLAN — customer-service-tool-agent

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

  API[SupportEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  Screener[ReplyScreener]:::cons
  WF[ConversationWorkflow]:::wf
  Agent[SupportAgent]:::agent
  TCGuard[ToolCallGuardrail]:::guard
  RGuard[ReplyGuardrail]:::guard
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|open + receiveMessage| Entity
  API -->|start workflow| WF
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-tool-call| TCGuard
  TCGuard -.->|allow / block| Agent
  Agent -.->|before-agent-response| RGuard
  RGuard -.->|accept / reject| Agent
  Agent -->|AgentReply| WF
  WF -->|recordReply| Entity
  WF -->|screenStep poll| Entity
  Entity -.->|AgentReplied| Screener
  Screener -->|recordScreened| Entity
  WF -->|escalateStep| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SupportEndpoint
  participant E as ConversationEntity
  participant W as ConversationWorkflow
  participant A as SupportAgent
  participant TC as ToolCallGuardrail
  participant RG as ReplyGuardrail
  participant S as ReplyScreener

  U->>API: POST /api/conversations
  API->>E: open(customerId)
  API->>E: receiveMessage(message)
  API->>W: start(conversationId)
  E-->>API: { conversationId }
  W->>E: markActive
  W->>A: runSingleTask(history + context)
  A->>TC: before-tool-call(lookupOrder)
  TC-->>A: allow
  A->>A: tool result: OrderRecord
  A->>RG: before-agent-response(candidate reply)
  RG-->>A: accept
  A-->>W: AgentReply
  W->>E: recordReply(reply)
  E-.->>S: AgentReplied
  S->>S: redact PII
  S->>E: recordScreened(screened)
  E-.->>U: SSE event(REPLIED)
```

## State machine — `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> ACTIVE: AgentProcessingStarted
  ACTIVE --> REPLIED: AgentReplied + ReplyScreened
  REPLIED --> ACTIVE: MessageReceived (follow-up)
  REPLIED --> RESOLVED: ConversationResolved
  ACTIVE --> ESCALATED: ConversationEscalated (tool blocked)
  OPEN --> FAILED: ConversationFailed (workflow error)
  ACTIVE --> FAILED: ConversationFailed (agent error)
  ESCALATED --> [*]
  RESOLVED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ ConversationOpened : emits
  ConversationEntity ||--o{ MessageReceived : emits
  ConversationEntity ||--o{ AgentProcessingStarted : emits
  ConversationEntity ||--o{ AgentReplied : emits
  ConversationEntity ||--o{ ReplyScreened : emits
  ConversationEntity ||--o{ ConversationEscalated : emits
  ConversationEntity ||--o{ ConversationResolved : emits
  ConversationEntity ||--o{ ConversationFailed : emits
  ConversationView }o--|| ConversationEntity : projects
  ReplyScreener }o--|| ConversationEntity : subscribes
  ConversationWorkflow }o--|| ConversationEntity : reads-and-writes
  SupportAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SupportEndpoint` | `api/SupportEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `ReplyScreener` | `application/ReplyScreener.java` |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `SupportAgent` | `application/SupportAgent.java` (tasks in `application/SupportTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `ConversationView` | `application/ConversationView.java` |
| `OrderTools` | `application/OrderTools.java` |
| `AccountTools` | `application/AccountTools.java` |
| `InventoryTools` | `application/InventoryTools.java` |
| `EscalationTools` | `application/EscalationTools.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `agentStep` 90 s, `screenStep` 15 s, `escalateStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ConversationWorkflow::error)`. The 90 s on `agentStep` accommodates LLM latency plus multiple tool-call round trips (Lesson 4).
- **Idempotency**: every workflow uses `"conv-" + conversationId + "-turn-" + turnIndex` as its id so multi-turn re-delivery is safe. `ConversationEntity.recordReply` is event-version-guarded — a duplicate call for the same turn is a no-op.
- **One agent per conversation**: the AutonomousAgent instance id is `"agent-" + conversationId`, giving each session its own conversation context. `maxIterationsPerTask(4)` caps guardrail-triggered retries.
- **G1 blocking**: when `ToolCallGuardrail` blocks a write call, the block result is returned as the tool's synthetic result. The agent reads the block message and can choose to escalate — no external rollback is needed because the tool never executed.
- **G2 retry**: when `ReplyGuardrail` rejects a candidate response, the agent loop counts one iteration. If all 4 iterations fail, `agentStep` fails over to `error` and the entity transitions to `FAILED`.
- **Screener is async**: `ReplyScreener` processes `AgentReplied` asynchronously. `screenStep` polls up to 15 s for `ReplyScreened` to land. This decouples PII log hygiene from the agent's reply latency.
- **No saga / no compensation**: within the auto-approve threshold, writes are idempotent (refund minted once per turn). Above threshold the write never executes — nothing to roll back.
