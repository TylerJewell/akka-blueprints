# PLAN — whatsapp-order-agent

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

  API[OrderEndpoint]:::ep
  SessionEnt[SessionEntity]:::ese
  OrderEnt[OrderEntity]:::ese
  Sanitizer[ConversationSanitizer]:::cons
  WF[OrderWorkflow]:::wf
  Agent[OrderAgent]:::agent
  Guard[ToolGuardrail]:::guard
  Scorer[TurnScorer]:::guard
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|startSession / receiveTurn| SessionEnt
  API -->|grantApproval / rejectApproval| SessionEnt
  SessionEnt -.->|TurnCompleted| Sanitizer
  Sanitizer -->|attachSanitizedTurn| SessionEnt
  WF -->|activateStep| SessionEnt
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|AgentReply| WF
  WF -->|recordTurnCompleted| SessionEnt
  WF -->|hitlStep poll| SessionEnt
  WF -->|completeStep score| Scorer
  Scorer -->|TurnEval| WF
  WF -->|recordTurnEval| SessionEnt
  WF -->|draft / confirm / cancel| OrderEnt
  SessionEnt -.->|projects| View
  OrderEnt -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, single-item order)

```mermaid
sequenceDiagram
  autonumber
  participant U as Operator (UI)
  participant API as OrderEndpoint
  participant SE as SessionEntity
  participant San as ConversationSanitizer
  participant WF as OrderWorkflow
  participant A as OrderAgent
  participant G as ToolGuardrail
  participant OE as OrderEntity
  participant Sc as TurnScorer

  U->>API: POST /api/sessions
  API->>SE: startSession(customerId)
  SE-->>API: { sessionId }
  U->>API: POST /api/sessions/{id}/turns
  API->>SE: receiveTurn(customerMessage, turnId)
  API->>WF: start agentStep
  WF->>A: runSingleTask(history + catalog attachment)
  A->>G: before-tool-call(create-order)
  G-->>A: allow
  A-->>WF: AgentReply (hitlRequired=false)
  WF->>SE: recordTurnCompleted(agentReply)
  WF->>OE: draft(orderRequest)
  WF->>OE: confirm
  SE-.->>San: TurnCompleted
  San->>San: redact PII
  San->>SE: attachSanitizedTurn
  WF->>Sc: score(agentReply, customerMessage)
  Sc-->>WF: TurnEval
  WF->>SE: recordTurnEval(eval)
  SE-.->>U: SSE event(COMPLETING)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> IDLE
  IDLE --> ACTIVE: SessionStarted
  ACTIVE --> ACTIVE: TurnCompleted (hitlRequired=false)
  ACTIVE --> AWAITING_APPROVAL: ApprovalRequested (hitlRequired=true)
  AWAITING_APPROVAL --> COMPLETING: ApprovalGranted
  AWAITING_APPROVAL --> FAILED: ApprovalRejected
  COMPLETING --> CLOSED: SessionCompleted
  ACTIVE --> FAILED: SessionFailed (agent error)
  CLOSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionStarted : emits
  SessionEntity ||--o{ TurnReceived : emits
  SessionEntity ||--o{ TurnCompleted : emits
  SessionEntity ||--o{ ConversationSanitized : emits
  SessionEntity ||--o{ ApprovalRequested : emits
  SessionEntity ||--o{ ApprovalGranted : emits
  SessionEntity ||--o{ ApprovalRejected : emits
  SessionEntity ||--o{ SessionCompleted : emits
  SessionEntity ||--o{ SessionFailed : emits
  OrderEntity ||--o{ OrderDrafted : emits
  OrderEntity ||--o{ OrderConfirmed : emits
  OrderEntity ||--o{ OrderCancelled : emits
  SessionView }o--|| SessionEntity : projects
  SessionView }o--|| OrderEntity : projects
  ConversationSanitizer }o--|| SessionEntity : subscribes
  OrderWorkflow }o--|| SessionEntity : reads-and-writes
  OrderWorkflow }o--|| OrderEntity : writes
  OrderAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `OrderEndpoint` | `api/OrderEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `OrderEntity` | `application/OrderEntity.java` (state in `domain/Order.java`, events in `domain/OrderEvent.java`) |
| `ConversationSanitizer` | `application/ConversationSanitizer.java` |
| `OrderWorkflow` | `application/OrderWorkflow.java` |
| `OrderAgent` | `application/OrderAgent.java` (tasks in `application/OrderTasks.java`) |
| `ToolGuardrail` | `application/ToolGuardrail.java` |
| `TurnScorer` | `application/TurnScorer.java` |
| `ProductCatalogService` | `application/ProductCatalogService.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `activateStep` 5 s, `agentStep` 60 s, `hitlStep` 300 s, `completeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(OrderWorkflow::error)`. The 60 s on `agentStep` accommodates LLM latency (Lesson 4). The 300 s on `hitlStep` gives operators a reasonable window to approve or reject.
- **Idempotency**: every workflow uses `"session-" + sessionId` as the workflow id. `ConversationSanitizer` is redelivery-safe — a second `TurnCompleted` for the same `turnId` is detected by the entity's applier as a no-op if `piiCategoriesRedacted` is already present.
- **One agent per session**: the AutonomousAgent instance id is `"agent-" + sessionId`, which gives each customer conversation its own context window. `maxIterationsPerTask(4)` caps guardrail-triggered re-plans at 4.
- **Guardrail-driven re-plan**: when `ToolGuardrail` blocks a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, `agentStep` fails over to `error` and the session entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `TurnScorer` runs in-process inside `completeStep`. No LLM call — the same reply always scores the same.
- **HITL serialisation**: the `hitlStep` polls via `SessionEntity.getSession` — it does not block a thread. The 300 s timeout is the maximum operator response window; if it expires, the workflow fails over to `error`.
- **No saga / no compensation**: order write operations are append-only events on `OrderEntity`. A cancelled order records `OrderCancelled`; there is nothing to roll back.
