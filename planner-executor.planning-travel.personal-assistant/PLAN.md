# PLAN — line-personal-assistant

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

  Planner[PlannerAgent]:::agent
  Cal[CalendarExecutorAgent]:::agent
  Gmail[GmailExecutorAgent]:::agent

  WF[ConversationWorkflow]:::wf
  Conv[ConversationEntity]:::ese
  Confirm[ConfirmationEntity]:::ese
  Queue[MessageQueue]:::ese
  View[ConversationView]:::view
  Consumer[MessageConsumer]:::cons
  Sim[MessageSimulator]:::ta
  Stale[StaleConversationMonitor]:::ta
  API[LineWebhookEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue| Queue
  API -->|resolve| Confirm
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_ACTION| Planner
  WF -->|EXECUTE_CALENDAR| Cal
  WF -->|EXECUTE_EMAIL| Gmail
  WF -->|emit events| Conv
  WF -->|open/resolve| Confirm
  Conv -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 60s| Conv
```

## Interaction sequence — J1/J2 (happy path with email confirmation)

```mermaid
sequenceDiagram
  autonumber
  participant U as LINE User
  participant API as LineWebhookEndpoint
  participant Q as MessageQueue
  participant C as MessageConsumer
  participant W as ConversationWorkflow
  participant P as PlannerAgent
  participant G as ActionPlanGuardrail
  participant CE as ConversationEntity
  participant CF as ConfirmationEntity
  participant EX as Executor (Calendar/Gmail)
  participant V as ConversationView

  U->>API: POST /api/webhook {text}
  API->>Q: append MessageEnqueued
  API-->>U: 200 {}
  Q->>C: MessageEnqueued
  C->>W: start({conversationId, text})
  W->>CE: emit ConversationCreated (PLANNING)
  W->>P: PLAN_ACTION(text)
  P-->>W: ActionPlan
  W->>G: vet(plan)
  alt plan passes guardrail
    alt intent = SEND_EMAIL
      W->>CE: emit ConfirmationRequested (AWAITING_CONFIRMATION)
      W->>CF: open confirmation
      W->>U: LINE confirm message
      U->>API: POST /webhook YES reply
      API->>CF: resolveConfirmation(approved=true)
      CF->>W: resume
      W->>CE: emit ConfirmationApproved (EXECUTING)
      W->>EX: EXECUTE_EMAIL(params)
      EX-->>W: ExecutionResult
    else intent = CREATE_EVENT / LIST_EVENTS
      W->>CE: emit PlanProduced (EXECUTING)
      W->>EX: EXECUTE_CALENDAR(params)
      EX-->>W: ExecutionResult
    end
    W->>CE: emit ActionExecuted (COMPLETED)
    W->>U: LINE reply with summary
    CE-->>V: project
    V-->>U: SSE update
  else plan blocked by guardrail
    W->>CE: emit ActionBlocked (BLOCKED)
    W->>U: LINE reply with block explanation
  end
```

## State machine — `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> AWAITING_CONFIRMATION: ConfirmationRequested
  PLANNING --> EXECUTING: PlanProduced (calendar intent)
  PLANNING --> BLOCKED: ActionBlocked
  PLANNING --> AWAITING_CLARIFICATION: ClarificationRequested
  AWAITING_CONFIRMATION --> EXECUTING: ConfirmationApproved
  AWAITING_CONFIRMATION --> CANCELLED: ConfirmationDenied
  AWAITING_CONFIRMATION --> EXPIRED: ConversationExpired
  AWAITING_CLARIFICATION --> PLANNING: new inbound message
  AWAITING_CLARIFICATION --> EXPIRED: ConversationExpired
  EXECUTING --> COMPLETED: ActionExecuted
  EXECUTING --> FAILED: ConversationFailed
  BLOCKED --> [*]
  CANCELLED --> [*]
  COMPLETED --> [*]
  EXPIRED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ ConversationCreated : emits
  ConversationEntity ||--o{ PlanProduced : emits
  ConversationEntity ||--o{ ActionBlocked : emits
  ConversationEntity ||--o{ ConfirmationRequested : emits
  ConversationEntity ||--o{ ConfirmationApproved : emits
  ConversationEntity ||--o{ ConfirmationDenied : emits
  ConversationEntity ||--o{ ActionExecuted : emits
  ConversationEntity ||--o{ ClarificationRequested : emits
  ConversationEntity ||--o{ ActionCancelled : emits
  ConversationEntity ||--o{ ConversationExpired : emits
  ConversationEntity ||--o{ ConversationFailed : emits
  ConversationView }o--|| ConversationEntity : projects
  ConfirmationEntity ||--o{ ConfirmationOpened : emits
  ConfirmationEntity ||--o{ ConfirmationResolved : emits
  ConfirmationEntity ||--o{ ConfirmationTimedOut : emits
  MessageQueue ||--o{ MessageEnqueued : emits
  MessageConsumer }o--|| MessageQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `CalendarExecutorAgent` | `application/CalendarExecutorAgent.java` |
| `GmailExecutorAgent` | `application/GmailExecutorAgent.java` |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `ConfirmationEntity` | `application/ConfirmationEntity.java` |
| `MessageQueue` | `application/MessageQueue.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MessageConsumer` | `application/MessageConsumer.java` |
| `MessageSimulator` | `application/MessageSimulator.java` |
| `StaleConversationMonitor` | `application/StaleConversationMonitor.java` |
| `ActionPlanGuardrail` | `application/ActionPlanGuardrail.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `LineWebhookEndpoint` | `api/LineWebhookEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 45 s, `executeStep` 60 s, `confirmRequestStep` 20 s, `replyStep` 20 s. Default recovery: `maxRetries(2).failoverTo(ConversationWorkflow::error)`.
- **Confirmation window:** `awaitConfirmStep` polls `ConfirmationEntity.get` every 3 s; the `StaleConversationMonitor` independently expires the entity at 10 minutes. The workflow exits on `APPROVED`, `DENIED`, or `EXPIRED`.
- **Halt-by-expiry:** `StaleConversationMonitor` ticks every 60 s and applies `expireConversation` to all conversations in `AWAITING_CONFIRMATION` or `AWAITING_CLARIFICATION` older than 10 minutes.
- **Idempotency:** `LineWebhookEndpoint` deduplicates on `X-Line-Message-Id` header; duplicate webhooks from the LINE platform are silently dropped.
- **Guardrail determinism:** `ActionPlanGuardrail.vet` is a pure function; it never inspects external state.
