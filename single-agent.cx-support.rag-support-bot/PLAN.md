# PLAN — rag-support-bot

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

  API[ConversationEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  KB[KnowledgeBase]:::ese
  Sanitizer[MessageSanitizer]:::cons
  WF[ConversationWorkflow]:::wf
  Agent[SupportAgent]:::agent
  Guard[ReplyGuardrail]:::guard
  Scorer[GroundingScorer]:::guard
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|receive| Entity
  Entity -.->|QueryReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|retrieveStep query| KB
  WF -->|recordRetrieval| Entity
  WF -->|replyStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|SupportReply| WF
  WF -->|recordReply| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|GroundingEval| WF
  WF -->|recordEvaluation| Entity
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
  participant S as MessageSanitizer
  participant W as ConversationWorkflow
  participant KB as KnowledgeBase
  participant A as SupportAgent
  participant G as ReplyGuardrail
  participant Sc as GroundingScorer

  U->>API: POST /api/conversations
  API->>E: receive(query)
  E-->>API: { conversationId }
  E-.->>S: QueryReceived
  S->>S: strip PII
  S->>E: attachSanitized
  S->>W: start(conversationId)
  W->>E: poll getConversation
  E-->>W: sanitized.isPresent()
  W->>KB: retrieve top-5 passages
  KB-->>W: RetrievalResult
  W->>E: recordRetrieval
  W->>E: markReplying
  W->>A: runSingleTask(query + passages attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: SupportReply
  W->>E: recordReply(reply)
  W->>Sc: score(reply, retrieved)
  Sc-->>W: GroundingEval
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: QuerySanitized
  SANITIZED --> RETRIEVING: RetrievalCompleted
  RETRIEVING --> REPLYING: ReplyStarted
  REPLYING --> REPLY_RECORDED: ReplyRecorded
  REPLY_RECORDED --> EVALUATED: EvaluationScored
  RECEIVED --> FAILED: ConversationFailed (sanitizer error)
  REPLYING --> FAILED: ConversationFailed (agent error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ QueryReceived : emits
  ConversationEntity ||--o{ QuerySanitized : emits
  ConversationEntity ||--o{ RetrievalCompleted : emits
  ConversationEntity ||--o{ ReplyStarted : emits
  ConversationEntity ||--o{ ReplyRecorded : emits
  ConversationEntity ||--o{ EvaluationScored : emits
  ConversationEntity ||--o{ ConversationFailed : emits
  KnowledgeBase ||--o{ KbPassage : contains
  ConversationView }o--|| ConversationEntity : projects
  MessageSanitizer }o--|| ConversationEntity : subscribes
  ConversationWorkflow }o--|| ConversationEntity : reads-and-writes
  ConversationWorkflow }o--|| KnowledgeBase : retrieves-from
  SupportAgent ||--o{ SupportReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversationEndpoint` | `api/ConversationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `KnowledgeBase` | `application/KnowledgeBase.java` (article record in `domain/KbArticle.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `SupportAgent` | `application/SupportAgent.java` (tasks in `application/SupportTasks.java`) |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `GroundingScorer` | `application/GroundingScorer.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `retrieveStep` 10 s, `replyStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ConversationWorkflow::error)`. The 60 s on `replyStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"conv-" + conversationId` as the workflow id; the `MessageSanitizer` Consumer is allowed to redeliver `QueryReceived` events because `ConversationEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized conversation is a no-op.
- **One agent per conversation**: the AutonomousAgent instance id is `"support-" + conversationId`, giving each task its own conversation context. `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ReplyGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. If all 3 iterations fail validation, the workflow's `replyStep` fails over to `error` and the entity transitions to `FAILED`.
- **Retrieval is in-process**: `retrieveStep` queries `KnowledgeBase` entity instances directly via `componentClient`. No external vector store — the baseline stays self-contained.
- **Eval is synchronous and deterministic**: `GroundingScorer` runs in-process inside `evalStep`. No LLM call — the same reply always scores the same. This is the single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
