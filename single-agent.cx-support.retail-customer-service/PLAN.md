# PLAN — retail-customer-service

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

  API[CustomerServiceEndpoint]:::ep
  SessionE[SessionEntity]:::ese
  OrderE[OrderEntity]:::ese
  WF[SessionWorkflow]:::wf
  Agent[CustomerServiceAgent]:::agent
  OGuard[OrderModificationGuardrail]:::guard
  RGuard[ReplyPolicyGuardrail]:::guard
  SessionV[SessionView]:::view
  OrderV[OrderView]:::view
  App[AppEndpoint]:::ep

  API -->|openSession / startTurn| SessionE
  API -->|getOrder| OrderE
  API -->|start workflow turn| WF
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-tool-call| OGuard
  Agent -.->|before-agent-response| RGuard
  Agent -->|AgentReply| WF
  WF -->|applyModificationStep| OrderE
  WF -->|replyStep recordTurnReply| SessionE
  SessionE -.->|projects| SessionV
  OrderE -.->|projects| OrderV
  API -->|list / SSE| SessionV
  API -->|order lookup| OrderV
  App -->|static| API
```

## Interaction sequence — J2 (order address-change happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Customer (UI)
  participant API as CustomerServiceEndpoint
  participant SE as SessionEntity
  participant WF as SessionWorkflow
  participant A as CustomerServiceAgent
  participant OG as OrderModificationGuardrail
  participant OE as OrderEntity
  participant RG as ReplyPolicyGuardrail

  U->>API: POST /api/sessions/{id}/turn {customerMessage, orderId}
  API->>SE: startTurn(turnId)
  API->>WF: start turn workflow
  WF->>SE: emit TurnStarted
  WF->>A: runSingleTask(context.json attachment)
  A->>OG: before-tool-call(updateOrderAddress, orderId)
  OG->>OE: getOrder(orderId)
  OE-->>OG: Order{status=PENDING}
  OG-->>A: allow
  A->>A: tool call updateOrderAddress
  A->>RG: before-agent-response(candidate reply)
  RG-->>A: accept
  A-->>WF: AgentReply{message, orderChange}
  WF->>OE: updateOrderAddress(orderId, newAddress)
  OE-->>WF: ok
  WF->>SE: recordTurnReply(turnId, reply)
  SE-.->>U: SSE event(TurnReplied)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> ACTIVE: TurnStarted
  ACTIVE --> ACTIVE: TurnReplied (next turn)
  ACTIVE --> ACTIVE: TurnBlockedByGuardrail
  ACTIVE --> ACTIVE: TurnFailed (recoverable)
  ACTIVE --> CLOSED: SessionClosed
  CLOSED --> [*]
```

## State machine — `OrderEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> PROCESSING: OrderProcessing
  PENDING --> CANCELLED: OrderCancelled
  PROCESSING --> SHIPPED: OrderShipped
  PROCESSING --> CANCELLED: OrderCancelled
  SHIPPED --> DELIVERED: OrderDelivered
  DELIVERED --> [*]
  CANCELLED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionOpened : emits
  SessionEntity ||--o{ TurnStarted : emits
  SessionEntity ||--o{ TurnReplied : emits
  SessionEntity ||--o{ TurnBlockedByGuardrail : emits
  SessionEntity ||--o{ TurnFailed : emits
  SessionEntity ||--o{ SessionClosed : emits
  OrderEntity ||--o{ OrderPlaced : emits
  OrderEntity ||--o{ OrderCancelled : emits
  OrderEntity ||--o{ OrderAddressUpdated : emits
  OrderEntity ||--o{ OrderQuantityUpdated : emits
  SessionView }o--|| SessionEntity : projects
  OrderView }o--|| OrderEntity : projects
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  SessionWorkflow }o--|| OrderEntity : writes-on-change
  CustomerServiceAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CustomerServiceEndpoint` | `api/CustomerServiceEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `OrderEntity` | `application/OrderEntity.java` (state in `domain/Order.java`, events in `domain/OrderEvent.java`) |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `CustomerServiceAgent` | `application/CustomerServiceAgent.java` (tasks in `application/CustomerServiceTasks.java`) |
| `OrderModificationGuardrail` | `application/OrderModificationGuardrail.java` |
| `ReplyPolicyGuardrail` | `application/ReplyPolicyGuardrail.java` |
| `SessionView` | `application/SessionView.java` |
| `OrderView` | `application/OrderView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `agentStep` 60 s, `applyModificationStep` 15 s, `replyStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SessionWorkflow::error)`. The 60 s on `agentStep` accommodates LLM latency plus up to 4 guardrail-retry iterations (Lesson 4).
- **Idempotency**: every workflow uses `"turn-" + sessionId + "-" + turnId` as the workflow id; duplicate `startTurn` calls are no-ops on the entity.
- **One agent per session**: the AutonomousAgent instance id is `"cs-" + sessionId`, giving each session its own conversation context across turns. The agent's `maxIterationsPerTask(4)` caps guardrail-triggered retries at 4 per turn.
- **Guardrail-driven retry**: when either guardrail rejects, the loop retries. If all 4 iterations exhaust on guardrail blocks, `agentStep` fails over to `error` and the turn transitions to `TurnFailed`.
- **Order entity as second-line defense**: modification commands on `OrderEntity` validate status independently of the guardrail. If a modification arrives directly (e.g., via the REST endpoint), the entity rejects it if the order is in a terminal state.
- **No saga / no compensation**: if `applyModificationStep` fails after `agentStep` succeeded, the workflow retries up to 2 times. If it ultimately fails, the turn is recorded as `TurnBlockedByGuardrail` so the agent's reply is still delivered — the modification simply did not apply, and the next turn can retry.
