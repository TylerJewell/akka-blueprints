# PLAN — memory-tool-router

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

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

  Router[RouterAgent]:::agent
  Memory[MemoryAgent]:::agent
  Calc[CalculatorAgent]:::agent
  KB[KnowledgeBaseAgent]:::agent
  Web[WebLookupAgent]:::agent
  Code[CodeRunnerAgent]:::agent

  WF[ConversationWorkflow]:::wf
  Conv[ConversationEntity]:::ese
  Mem[MemoryEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[MessageQueue]:::ese
  View[ConversationView]:::view
  Consumer[MessageQueueConsumer]:::cons
  Sim[SessionSimulator]:::ta
  Stuck[StuckConversationMonitor]:::ta
  API[ConversationEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start/resume workflow| WF
  WF -->|RECALL / EXTRACT_MEMORIES| Memory
  WF -->|CLASSIFY_TURN / REVISE_TURN| Router
  WF -->|EVALUATE| Calc
  WF -->|LOOK_UP| KB
  WF -->|WEB_FETCH| Web
  WF -->|RUN_EXPRESSION| Code
  WF -->|emit events| Conv
  WF -->|read/write items| Mem
  WF -->|poll| Ctrl
  Conv -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Conv
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ConversationEndpoint
  participant Q as MessageQueue
  participant C as MessageQueueConsumer
  participant W as ConversationWorkflow
  participant MA as MemoryAgent
  participant R as RouterAgent
  participant T as ToolAgent (Calculator/KB/Web/Code)
  participant CE as ConversationEntity
  participant ME as MemoryEntity
  participant CTL as SystemControlEntity
  participant V as ConversationView

  U->>API: POST /api/conversations {sessionId, text}
  API->>Q: append MessageSubmitted
  API-->>U: 202 {conversationId}
  Q->>C: MessageSubmitted
  C->>W: start({conversationId, sessionId, text})
  W->>CE: emit ConversationStarted + MessageReceived (PROCESSING)
  W->>MA: RECALL(sessionId)
  MA-->>W: MemoryContext
  W->>R: CLASSIFY_TURN(text, memoryContext)
  R-->>W: TurnPlan
  W->>W: guardrail.vet(routingDecision)
  W->>T: runSingleTask(toolQuery)
  T-->>W: ToolResult
  W->>MA: EXTRACT_MEMORIES(text, toolResult)
  MA-->>W: MemoryDelta
  W->>W: PiiScrubber.scrub(memoryDelta.items)
  W->>ME: addItems(scrubbedItems)
  W->>CE: emit TurnCompleted
  W->>CTL: get halt flag
  CTL-->>W: halted=false
  CE-->>V: project
  V-->>U: SSE update
```

## State machine — `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> IDLE
  IDLE --> PROCESSING: MessageReceived
  PROCESSING --> PROCESSING: TurnCompleted / TurnBlocked / TurnFailed
  PROCESSING --> HALTED: ConversationHaltedOperator / ConversationHaltedAutomatic
  PROCESSING --> STUCK: ConversationFailedTimeout
  HALTED --> PROCESSING: MessageReceived (after Resume)
  STUCK --> [*]
  ENDED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ ConversationStarted : emits
  ConversationEntity ||--o{ MessageReceived : emits
  ConversationEntity ||--o{ TurnStarted : emits
  ConversationEntity ||--o{ TurnBlocked : emits
  ConversationEntity ||--o{ TurnCompleted : emits
  ConversationEntity ||--o{ TurnFailed : emits
  ConversationEntity ||--o{ ConversationHaltedOperator : emits
  ConversationEntity ||--o{ ConversationHaltedAutomatic : emits
  ConversationEntity ||--o{ ConversationFailedTimeout : emits
  ConversationView }o--|| ConversationEntity : projects
  MemoryEntity ||--o{ MemoryInitialised : emits
  MemoryEntity ||--o{ MemoryItemsAdded : emits
  MemoryEntity ||--o{ MemoryItemsForgotten : emits
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  MessageQueue ||--o{ MessageSubmitted : emits
  MessageQueueConsumer }o--|| MessageQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RouterAgent` | `application/RouterAgent.java` |
| `MemoryAgent` | `application/MemoryAgent.java` |
| `CalculatorAgent` | `application/CalculatorAgent.java` |
| `KnowledgeBaseAgent` | `application/KnowledgeBaseAgent.java` |
| `WebLookupAgent` | `application/WebLookupAgent.java` |
| `CodeRunnerAgent` | `application/CodeRunnerAgent.java` |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `MemoryEntity` | `application/MemoryEntity.java` (state in `domain/MemoryStore.java`, events in `domain/MemoryEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `MessageQueue` | `application/MessageQueue.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MessageQueueConsumer` | `application/MessageQueueConsumer.java` |
| `SessionSimulator` | `application/SessionSimulator.java` |
| `StuckConversationMonitor` | `application/StuckConversationMonitor.java` |
| `ToolDispatchGuardrail` | `application/ToolDispatchGuardrail.java` |
| `PiiScrubber` | `application/PiiScrubber.java` |
| `RouterTasks` | `application/RouterTasks.java` |
| `MemoryTasks` | `application/MemoryTasks.java` |
| `ToolTasks` | `application/ToolTasks.java` |
| `ConversationEndpoint` | `api/ConversationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `recallStep` 30 s, `routeStep` 45 s, `executeStep` 90 s (covers any tool call including a slow LLM), `extractMemoriesStep` 30 s, `replyStep` 30 s. Default recovery: `maxRetries(2).failoverTo(ConversationWorkflow::error)`.
- **Tool routing:** the workflow's `executeStep` switches on `RoutingDecision.tool`; `NONE` skips execution entirely and falls through to `extractMemoriesStep` with an empty `ToolResult`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. A halt arriving during `executeStep` lets the in-flight tool call finish; the loop exits at the next `checkHaltStep`.
- **Memory isolation:** `MemoryEntity` is keyed by `sessionId`, not `conversationId`. Two conversations in the same session share the same memory store; different sessions are fully isolated.
- **PII scrubber determinism:** `PiiScrubber.scrub` is pure; it never inspects external state. The same input always yields the same scrubbed output, keeping `MemoryItemsAdded` events deterministic and replayable.
- **Idempotency:** `ConversationEndpoint.submit` deduplicates on `(sessionId, text)` over a 10 s window.
- **Stuck detection:** `StuckConversationMonitor` ticks every 30 s; conversations `PROCESSING` for > 4 minutes are marked `STUCK`. The workflow's next step reads the status and exits.
