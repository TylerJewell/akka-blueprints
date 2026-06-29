# PLAN — guardrails-demo

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

  API[SessionEndpoint]:::ep
  Entity[SessionEntity]:::ese
  WF[SessionWorkflow]:::wf
  Agent[ConversationAgent]:::agent
  InGuard[TopicPolicyGuardrail]:::guard
  OutGuard[ContentPolicyGuardrail]:::guard
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|open session| Entity
  API -->|send turn| WF
  WF -->|openStep| Entity
  WF -->|exchangeStep runSingleTask| Agent
  Agent -.->|before-llm-call| InGuard
  InGuard -.->|topic-blocked| WF
  Agent -.->|before-agent-response| OutGuard
  OutGuard -.->|content-rejected| Agent
  Agent -->|AgentReply| WF
  WF -->|recordReply| Entity
  WF -->|recordBlock| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SessionEndpoint
  participant E as SessionEntity
  participant W as SessionWorkflow
  participant IG as TopicPolicyGuardrail
  participant A as ConversationAgent
  participant OG as ContentPolicyGuardrail

  U->>API: POST /api/sessions/{id}/turns
  API->>W: triggerExchangeStep(turnId, userMessage)
  W->>E: receiveTurn(request)
  W->>IG: before-llm-call check
  IG-->>W: ALLOWED
  W->>E: recordTopicCheck(ALLOWED)
  W->>E: markGenerating(turnId)
  W->>A: runSingleTask(userMessage + context)
  A->>OG: before-agent-response(candidate)
  OG-->>A: pass
  A-->>W: AgentReply
  W->>E: recordReply(turnId, reply)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `SessionEntity` (per-turn view)

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> CHECKING_INPUT: TurnReceived
  CHECKING_INPUT --> BLOCKED: TurnBlocked (topic matched)
  CHECKING_INPUT --> GENERATING: GenerationStarted
  GENERATING --> COMPLETED: ReplyRecorded
  GENERATING --> FAILED: TurnFailed (agent error)
  BLOCKED --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionOpened : emits
  SessionEntity ||--o{ TurnReceived : emits
  SessionEntity ||--o{ TopicChecked : emits
  SessionEntity ||--o{ TurnBlocked : emits
  SessionEntity ||--o{ GenerationStarted : emits
  SessionEntity ||--o{ ReplyRecorded : emits
  SessionEntity ||--o{ TurnFailed : emits
  SessionEntity ||--o{ SessionClosed : emits
  SessionView }o--|| SessionEntity : projects
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  ConversationAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `ConversationAgent` | `application/ConversationAgent.java` (tasks in `application/SessionTasks.java`) |
| `TopicPolicyGuardrail` | `application/TopicPolicyGuardrail.java` |
| `ContentPolicyGuardrail` | `application/ContentPolicyGuardrail.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `openStep` 5 s, `exchangeStep` 60 s, `closeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SessionWorkflow::error)`. The 60 s on `exchangeStep` accommodates LLM latency plus up to 3 before-agent-response iterations (Lesson 4).
- **Idempotency**: the workflow id is `"session-" + sessionId`. Re-triggering `exchangeStep` for the same `turnId` is event-version-guarded on the entity — a re-delivered `TurnReceived` for an already-processed turn is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"agent-" + sessionId`. All turns within a session share the same agent instance, preserving conversational context. The `capability(...).maxIterationsPerTask(3)` caps content-policy retries at 3.
- **Before-llm-call intercept**: when `TopicPolicyGuardrail` fires, no model call is made. The rejection propagates from the guardrail hook back to `exchangeStep`, which records `TurnBlocked` directly without entering the agent's iteration loop.
- **Content-policy retry**: when `ContentPolicyGuardrail` rejects a candidate reply, the rejection returns to the agent loop. Each rejection consumes one iteration from the `maxIterationsPerTask(3)` budget. If all 3 iterations fail, `exchangeStep` fails over to `error` and the entity records `TurnFailed`.
- **No saga / no compensation**: turns are append-only; a failed turn leaves prior turns intact. The session remains `OPEN` after a `TurnFailed` so the user can continue with a new message.
