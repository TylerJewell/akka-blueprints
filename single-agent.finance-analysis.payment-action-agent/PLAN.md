# PLAN — antom-payment

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

  API[PaymentEndpoint]:::ep
  Entity[PaymentEntity]:::ese
  FraudConsumer[FraudSignalConsumer]:::cons
  WF[PaymentWorkflow]:::wf
  Agent[PaymentActionAgent]:::agent
  Guard[AuthorizationGuardrail]:::guard
  Sim[AntomApiSimulator]:::guard
  View[PaymentView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|approve/deny| WF
  API -->|start workflow| WF
  Entity -.->|FraudSignalDetected| FraudConsumer
  FraudConsumer -->|halt| Entity
  WF -->|authStep check| Guard
  Guard -->|pass/reject| WF
  WF -->|authorize/reject| Entity
  WF -->|hitlGateStep pause/resume| Entity
  WF -->|executeStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|tool calls| Sim
  Sim -->|AntomApiResponse| Agent
  Agent -->|PaymentResult| WF
  WF -->|settle/query| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, sub-threshold)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as PaymentEndpoint
  participant E as PaymentEntity
  participant W as PaymentWorkflow
  participant G as AuthorizationGuardrail
  participant A as PaymentActionAgent
  participant S as AntomApiSimulator

  U->>API: POST /api/payments
  API->>E: submit(instruction)
  E-->>API: { paymentId }
  API->>W: start(paymentId)
  W->>G: authStep check(instruction)
  G-->>W: pass
  W->>E: authorize()
  W->>W: hitlGateStep (below threshold)
  W->>E: startExecution()
  W->>A: runSingleTask(instruction)
  A->>G: before-tool-call(initiatePayment)
  G-->>A: pass
  A->>S: initiatePayment(instruction)
  S-->>A: AntomApiResponse
  A-->>W: PaymentResult
  W->>E: settle(result)
  E-.->>U: SSE event(SETTLED)
```

## State machine — `PaymentEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> AUTHORIZED: PaymentAuthorized
  REQUESTED --> REJECTED: PaymentRejected (guardrail fail)
  AUTHORIZED --> AWAITING_APPROVAL: ApprovalRequired (high-value)
  AUTHORIZED --> EXECUTING: ExecutionStarted (sub-threshold)
  AWAITING_APPROVAL --> EXECUTING: ApprovalGranted
  AWAITING_APPROVAL --> REJECTED: ApprovalDenied
  EXECUTING --> SETTLED: PaymentSettled
  EXECUTING --> QUERIED: PaymentQueried
  EXECUTING --> HALTED: PaymentHalted (fraud signal)
  EXECUTING --> FAILED: PaymentFailed (agent error)
  SETTLED --> [*]
  QUERIED --> [*]
  HALTED --> [*]
  REJECTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PaymentEntity ||--o{ PaymentRequested : emits
  PaymentEntity ||--o{ PaymentAuthorized : emits
  PaymentEntity ||--o{ PaymentRejected : emits
  PaymentEntity ||--o{ ApprovalRequired : emits
  PaymentEntity ||--o{ ApprovalGranted : emits
  PaymentEntity ||--o{ ApprovalDenied : emits
  PaymentEntity ||--o{ ExecutionStarted : emits
  PaymentEntity ||--o{ PaymentSettled : emits
  PaymentEntity ||--o{ PaymentQueried : emits
  PaymentEntity ||--o{ FraudSignalDetected : emits
  PaymentEntity ||--o{ PaymentHalted : emits
  PaymentEntity ||--o{ PaymentFailed : emits
  PaymentView }o--|| PaymentEntity : projects
  FraudSignalConsumer }o--|| PaymentEntity : subscribes
  PaymentWorkflow }o--|| PaymentEntity : reads-and-writes
  PaymentActionAgent ||--o{ PaymentResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PaymentEndpoint` | `api/PaymentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PaymentEntity` | `application/PaymentEntity.java` (state in `domain/Payment.java`, events in `domain/PaymentEvent.java`) |
| `FraudSignalConsumer` | `application/FraudSignalConsumer.java` |
| `PaymentWorkflow` | `application/PaymentWorkflow.java` |
| `PaymentActionAgent` | `application/PaymentActionAgent.java` (tasks in `application/PaymentTasks.java`) |
| `AuthorizationGuardrail` | `application/AuthorizationGuardrail.java` |
| `AntomApiSimulator` | `application/AntomApiSimulator.java` |
| `PaymentView` | `application/PaymentView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `authStep` 5 s, `hitlGateStep` 86 400 s (24 h operator window), `executeStep` 60 s, `recordStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(PaymentWorkflow::error)`. The 60 s on `executeStep` accommodates LLM latency plus the simulated Antom API call (Lesson 4).
- **Idempotency**: every workflow uses `"payment-" + paymentId` as the workflow id. `PaymentEntity.authorize` is event-version-guarded — a second authorization attempt on an already-authorized payment is a no-op.
- **One agent per payment**: the AutonomousAgent instance id is `"agent-" + paymentId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps retries.
- **Guardrail-driven rejection**: when `AuthorizationGuardrail` rejects a tool call in the before-tool-call hook, the rejection is returned as a structured `authorization-failure` to the agent loop. The agent logs the block and does not reattempt the same tool call.
- **Halt is unconditional**: `FraudSignalConsumer` fires on any `FraudSignalDetected` event regardless of workflow step. The entity transitions to `HALTED` and any in-progress workflow step fails fast on the next tick.
- **HITL pause**: `hitlGateStep` uses `workflow.pause()` to suspend the workflow thread. The entity is in `AWAITING_APPROVAL`. `PaymentEndpoint.approve` / `.deny` call `workflow.resume(ApprovalGranted)` / `workflow.resume(ApprovalDenied)` to unblock the step.
- **No saga / no compensation**: payment tool calls are simulated; there is nothing external to roll back. A real deployer adding live Antom API calls must add a compensation step to `error` that issues a reversal if `ExecutionStarted` was already emitted.
