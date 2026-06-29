# PLAN — akka-async-agent-mesh

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab with the Akka theme variables and the Lesson 24 state-label CSS overrides.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef kve fill:#1f1900,stroke:#C9A227,color:#C9A227;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Dispatcher[DispatcherAgent]:::agent
  Processor[ProcessorAgent]:::agent

  DispWF[MessageDispatchWorkflow]:::wf
  ProcWF[MessageProcessingWorkflow]:::wf

  MsgE[MessageEntity]:::ese
  MemE[AgentMemoryEntity]:::ese
  RcptE[DeliveryReceiptEntity]:::ese
  Bus[MessageBusEntity]:::ese
  Control[SystemControl]:::kve
  BusView[MessageBusView]:::view
  Consumer[MessageBusConsumer]:::cons
  Sim[ScenarioSimulator]:::ta
  Stale[StaleMessageMonitor]:::ta
  API[MeshEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit dispatch| Bus
  Sim -.->|every 60s| Bus
  Bus -.->|subscribes| Consumer
  Consumer -->|start workflow| DispWF
  DispWF -->|call| Dispatcher
  DispWF -->|create message| MsgE
  DispWF -->|record receipt| RcptE
  MsgE -.->|projects| BusView
  MsgE -.->|subscribes| ProcWF
  ProcWF -->|read context| MemE
  ProcWF -->|call| Processor
  ProcWF -->|update memory| MemE
  ProcWF -->|complete message| MsgE
  ProcWF -->|read flag| Control
  Stale -.->|every 90s| MsgE
  API -->|halt / resume| Control
  API -->|query / SSE| BusView
  API -->|read memory| MemE
  API -->|read receipt| RcptE
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. `ProcessorAgent` is one agent class run as two instances (`agent-alpha`, `agent-beta`); each instance processes messages addressed to it via its own `MessageProcessingWorkflow`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as MeshEndpoint
  participant Bus as MessageBusEntity
  participant C as MessageBusConsumer
  participant DW as MessageDispatchWorkflow
  participant D as DispatcherAgent
  participant ME as MessageEntity
  participant PW as MessageProcessingWorkflow
  participant P as ProcessorAgent
  participant Mem as AgentMemoryEntity
  participant V as MessageBusView

  U->>API: POST /api/dispatch {fromAgent, toAgent, topic, body}
  API->>Bus: submitDispatch(request)
  API-->>U: 202 {requestId}
  Bus->>C: DispatchRequestSubmitted
  C->>DW: start({requestId})
  DW->>D: compose(request)
  Note over D: every send_message_to_agent_async call passes the before-tool-call guardrail
  D-->>DW: DispatchResult{messageId, receiptId}
  DW->>ME: createMessage (status DISPATCHED)
  ME-->>V: MessageBusView row
  DW->>ME: deliver → DELIVERED
  Note over PW: MessageBusConsumer starts processing workflow on MessageDispatched
  PW->>PW: sanitizeStep (PiiSanitizer.sanitize)
  PW->>ME: startProcessing → PROCESSING
  PW->>Mem: getMemory(toAgent)
  PW->>P: process(message, memory)
  P-->>PW: ProcessingResult{summary, newMemory}
  PW->>Mem: addEntries(newMemory)
  PW->>ME: complete(summary) → PROCESSED
  ME-->>V: row PROCESSED
  V-->>U: SSE update
```

## State machine — `MessageEntity`

```mermaid
stateDiagram-v2
  [*] --> DISPATCHED
  DISPATCHED --> DELIVERED: deliver (receipt recorded)
  DISPATCHED --> SEND_BLOCKED: blockSend (guardrail refusal)
  DISPATCHED --> STALE: stale (StaleMessageMonitor > 3 min)
  DELIVERED --> PROCESSING: startProcessing
  PROCESSING --> PROCESSED: complete (summary written)
  SEND_BLOCKED --> [*]
  STALE --> [*]
  PROCESSED --> [*]
```

## Entity model

```mermaid
erDiagram
  MessageEntity ||--o{ MessageDispatched : emits
  MessageEntity ||--o{ MessageDelivered : emits
  MessageEntity ||--o{ MessageProcessingStarted : emits
  MessageEntity ||--o{ MessageProcessed : emits
  MessageEntity ||--o{ MessageSendBlocked : emits
  MessageEntity ||--o{ MessageStaled : emits
  MessageBusView }o--|| MessageEntity : projects
  AgentMemoryEntity ||--o{ MemoryEntryAdded : emits
  AgentMemoryEntity ||--o{ MemoryCleared : emits
  MessageBusEntity ||--o{ DispatchRequestSubmitted : emits
  MessageBusConsumer }o--|| MessageBusEntity : subscribes
  DeliveryReceiptEntity ||--o{ ReceiptRecorded : emits
  MessageEntity }o--|| DeliveryReceiptEntity : "receipt per message"
  MessageEntity }o--|| AgentMemoryEntity : "updates memory"
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DispatcherAgent` | `application/DispatcherAgent.java` |
| `ProcessorAgent` | `application/ProcessorAgent.java` |
| `MeshTasks` | `application/MeshTasks.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `MessageDispatchWorkflow` | `application/MessageDispatchWorkflow.java` |
| `MessageProcessingWorkflow` | `application/MessageProcessingWorkflow.java` |
| `MessageEntity` | `application/MessageEntity.java` (state in `domain/Message.java`, events in `domain/MessageEvent.java`) |
| `AgentMemoryEntity` | `application/AgentMemoryEntity.java` (state in `domain/AgentMemory.java`, events in `domain/AgentMemoryEvent.java`) |
| `DeliveryReceiptEntity` | `application/DeliveryReceiptEntity.java` |
| `MessageBusEntity` | `application/MessageBusEntity.java` |
| `SystemControl` | `application/SystemControl.java` |
| `MessageBusView` | `application/MessageBusView.java` |
| `MessageBusConsumer` | `application/MessageBusConsumer.java` |
| `ScenarioSimulator` | `application/ScenarioSimulator.java` |
| `StaleMessageMonitor` | `application/StaleMessageMonitor.java` |
| `MeshEndpoint` | `api/MeshEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **2 autonomous-agent · 2 workflow · 4 event-sourced-entity · 1 key-value-entity · 1 view · 1 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Async fire-and-forget is the whole pattern.** `MessageDispatchWorkflow` terminates as soon as the delivery receipt is recorded — it does not wait for `MessageProcessingWorkflow` to start or finish. The two workflows run in parallel, connected only by `MessageEntity` events that `MessageBusConsumer` turns into new workflow starts.
- **Workflow step timeouts:** `MessageDispatchWorkflow.composeStep` and `MessageProcessingWorkflow.processStep` call agents, so each sets an explicit `stepTimeout` of 90 s (Lesson 4). The default 5 s timeout would expire mid-LLM-call.
- **PII sanitizer fires before any write.** `MessageProcessingWorkflow.sanitizeStep` runs before `processStep` and before any call to `AgentMemoryEntity` or `MessageEntity.complete`, so no unsanitized payload reaches durable storage.
- **Before-tool-call guardrail reads the roster.** The G1 guardrail on `DispatcherAgent` reads `async-mesh.agents` from `application.conf` at startup; a recipient not in that list is refused before the `send_message_to_agent_async` tool executes, and the request is recorded `SEND_BLOCKED`.
- **Halt:** `SystemControl` is read at the top of `MessageProcessingWorkflow.receiveStep`, so a halt stops new processing starts without affecting the dispatch side that has already recorded its receipt.
- **Stale detection:** `StaleMessageMonitor` fires every 90 s and advances `DISPATCHED` messages older than 3 minutes to `STALE`, so the board never accumulates orphaned rows from failed workflows.
- **Memory isolation:** each agent instance (`agent-alpha`, `agent-beta`) has its own `AgentMemoryEntity` keyed by agent id, so the long-term memory of one agent cannot contaminate the other's context.
