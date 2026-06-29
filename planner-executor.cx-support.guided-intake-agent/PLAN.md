# PLAN — guided-intake-agent

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

  Intake[IntakeAgent]:::agent
  Sanitizer[SanitizerAgent]:::agent

  WF[ConversationWorkflow]:::wf
  Conv[ConversationEntity]:::ese
  Catalog[GoalCatalogEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[SessionQueue]:::ese
  View[ConversationView]:::view
  Consumer[ConversationRequestConsumer]:::cons
  Sim[SessionSimulator]:::ta
  Stale[StaleSessionMonitor]:::ta
  API[ConversationEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|start session| Queue
  API -->|reply| WF
  API -->|halt/resume| Ctrl
  API -->|admin goals| Catalog
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_INTAKE / PROPOSE / EVALUATE / COMPOSE| Intake
  WF -->|SANITIZE_REPLY| Sanitizer
  WF -->|emit events| Conv
  WF -->|poll| Ctrl
  WF -->|read goal| Catalog
  Conv -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 60s| Conv
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ConversationEndpoint
  participant Q as SessionQueue
  participant C as ConversationRequestConsumer
  participant W as ConversationWorkflow
  participant I as IntakeAgent
  participant S as SanitizerAgent
  participant E as ConversationEntity
  participant CTL as SystemControlEntity
  participant V as ConversationView

  U->>API: POST /api/conversations {goalId}
  API->>Q: append SessionSubmitted
  API-->>U: 202 {conversationId}
  Q->>C: SessionSubmitted
  C->>W: start({conversationId, goalId})
  W->>E: emit ConversationCreated (PLANNING)
  W->>I: PLAN_INTAKE(goalDefinition)
  I-->>W: IntakePlan
  W->>E: emit ConversationPlanned, status ELICITING
  loop until GoalMet | Escalate | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>I: PROPOSE_QUESTION(plan, turns)
    I-->>W: QuestionProposal
    W->>W: guardrail.vet(proposal, goal)
    W->>E: emit QuestionAsked
    W-->>U: SSE question text
    U->>API: POST /api/conversations/{id}/reply
    API->>W: resume(reply)
    W->>S: SANITIZE_REPLY(rawReply)
    S-->>W: SanitizedReply
    W->>E: emit TurnRecorded
    W->>I: EVALUATE_REPLY(turns)
  end
  W->>I: COMPOSE_SUMMARY
  I-->>W: IntakeSummary
  W->>E: emit ConversationCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> ELICITING: ConversationPlanned
  ELICITING --> ELICITING: QuestionAsked / QuestionBlocked / TurnRecorded / PlanRevised
  ELICITING --> COMPLETED: ConversationCompleted
  ELICITING --> ESCALATED: ConversationEscalated
  ELICITING --> HALTED: ConversationHaltedOperator
  ELICITING --> ABANDONED: ConversationAbandoned
  COMPLETED --> [*]
  ESCALATED --> [*]
  HALTED --> [*]
  ABANDONED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ ConversationCreated : emits
  ConversationEntity ||--o{ ConversationPlanned : emits
  ConversationEntity ||--o{ QuestionAsked : emits
  ConversationEntity ||--o{ QuestionBlocked : emits
  ConversationEntity ||--o{ TurnRecorded : emits
  ConversationEntity ||--o{ PlanRevised : emits
  ConversationEntity ||--o{ ConversationCompleted : emits
  ConversationEntity ||--o{ ConversationEscalated : emits
  ConversationEntity ||--o{ ConversationHaltedOperator : emits
  ConversationEntity ||--o{ ConversationAbandoned : emits
  ConversationView }o--|| ConversationEntity : projects
  GoalCatalogEntity ||--o{ GoalAdded : emits
  GoalCatalogEntity ||--o{ GoalUpdated : emits
  GoalCatalogEntity ||--o{ GoalRemoved : emits
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  SessionQueue ||--o{ SessionSubmitted : emits
  ConversationRequestConsumer }o--|| SessionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `IntakeAgent` | `application/IntakeAgent.java` |
| `SanitizerAgent` | `application/SanitizerAgent.java` |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `GoalCatalogEntity` | `application/GoalCatalogEntity.java` |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `SessionQueue` | `application/SessionQueue.java` |
| `ConversationView` | `application/ConversationView.java` |
| `ConversationRequestConsumer` | `application/ConversationRequestConsumer.java` |
| `SessionSimulator` | `application/SessionSimulator.java` |
| `StaleSessionMonitor` | `application/StaleSessionMonitor.java` |
| `TopicGuardrail` | `application/TopicGuardrail.java` |
| `PiiScrubber` | `application/PiiScrubber.java` |
| `IntakeTasks` | `application/IntakeTasks.java` |
| `SanitizerTasks` | `application/SanitizerTasks.java` |
| `ConversationEndpoint` | `api/ConversationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `awaitReplyStep` 300 s (user has up to 5 min), `sanitizeStep` 30 s, `evaluateStep` 45 s, `summaryStep` 60 s. Default recovery: `maxRetries(2).failoverTo(ConversationWorkflow::error)`.
- **Replan budget:** the agent may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Escalate`.
- **Turn budget:** each goal specifies `maxTurns`; when the turn log reaches that count the workflow transitions to `escalateStep` even if the goal is not yet met.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during `awaitReplyStep` lets the in-flight reply land; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `ConversationEndpoint.start` uses `(goalId, submittedBy)` over a 10 s window to dedupe concurrent submissions.
- **Stale detection:** `StaleSessionMonitor` ticks every 60 s; conversations `ELICITING` for > 10 minutes are marked `ABANDONED`. The workflow's `evaluateStep` checks the entity status and exits when it reads `ABANDONED`.
- **PII scrubber determinism:** `PiiScrubber.scrub` is pure; given the same input it always produces the same output. Raw replies are never stored in any event.
