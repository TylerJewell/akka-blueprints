# PLAN — realtime-conversational-agent

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
  classDef scorer fill:#251503,stroke:#F97316,color:#F97316;

  API[SessionEndpoint]:::ep
  Entity[SessionEntity]:::ese
  WF[SessionWorkflow]:::wf
  Agent[ConversationalAgent]:::agent
  Guard[ResponseGuardrail]:::guard
  Summarizer[TurnSummarizer]:::scorer
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|open| Entity
  API -->|startWorkflow| WF
  API -->|submitTurn| WF
  API -->|requestEnd| WF
  WF -->|greetStep runSingleTask| Agent
  WF -->|converseTurnStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|pass/reject| Agent
  Agent -->|AgentTurn| WF
  WF -->|recordGreeting| Entity
  WF -->|receiveCustomerTurn| Entity
  WF -->|recordAgentReply| Entity
  WF -->|summarizeStep| Summarizer
  Summarizer -->|SessionSummary| WF
  WF -->|recordSummary| Entity
  Entity -.->|projects| View
  API -->|list/get/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, 2-turn session)

```mermaid
sequenceDiagram
  autonumber
  participant U as Customer (UI)
  participant API as SessionEndpoint
  participant E as SessionEntity
  participant W as SessionWorkflow
  participant A as ConversationalAgent
  participant G as ResponseGuardrail
  participant Sm as TurnSummarizer

  U->>API: POST /api/sessions
  API->>E: open(customerId)
  E-->>API: { sessionId }
  API->>W: start(sessionId)
  W->>A: greetStep runSingleTask(GREET_CUSTOMER)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: AgentTurn (greeting)
  W->>E: recordGreeting(agentTurn)
  E-.->>U: SSE event(ACTIVE)
  U->>API: POST /api/sessions/{id}/turns
  API->>W: submitTurn(customerMessage)
  W->>E: receiveCustomerTurn
  W->>A: converseTurnStep runSingleTask(attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: AgentTurn (reply)
  W->>E: recordAgentReply(turnId, agentTurn)
  E-.->>U: SSE event(ACTIVE + turn)
  U->>API: POST /api/sessions/{id}/end
  API->>W: requestEnd
  W->>Sm: summarize(turns)
  Sm-->>W: SessionSummary
  W->>E: recordSummary(summary)
  E-.->>U: SSE event(CLOSED)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> GREETING
  GREETING --> ACTIVE: GreetingEmitted
  ACTIVE --> WAITING_FOR_AGENT: CustomerTurnReceived
  WAITING_FOR_AGENT --> ACTIVE: AgentReplied
  WAITING_FOR_AGENT --> FAILED: TurnFailed (guardrail exhausted)
  ACTIVE --> SUMMARIZING: SessionEndRequested
  SUMMARIZING --> CLOSED: SessionSummarized
  GREETING --> FAILED: SessionFailed (greet error)
  SUMMARIZING --> FAILED: SessionFailed (summarize error)
  CLOSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionOpened : emits
  SessionEntity ||--o{ GreetingEmitted : emits
  SessionEntity ||--o{ CustomerTurnReceived : emits
  SessionEntity ||--o{ AgentReplied : emits
  SessionEntity ||--o{ TurnFailed : emits
  SessionEntity ||--o{ SessionEndRequested : emits
  SessionEntity ||--o{ SessionSummarized : emits
  SessionEntity ||--o{ SessionFailed : emits
  SessionView }o--|| SessionEntity : projects
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  ConversationalAgent ||--o{ AgentTurn : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `ConversationalAgent` | `application/ConversationalAgent.java` (tasks in `application/ConversationTasks.java`) |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `TurnSummarizer` | `application/TurnSummarizer.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `greetStep` 15 s, `converseTurnStep` 60 s, `summarizeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SessionWorkflow::error)`. The 60 s on `converseTurnStep` accommodates real-time LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"session-" + sessionId` as the workflow id; `SessionEndpoint.submitTurn` mints a new `turnId` per request; duplicate POSTs to the same turn endpoint produce a no-op if the turn is already in `AGENT_REPLIED` state.
- **One agent per session**: the AutonomousAgent instance id is `"agent-" + sessionId`, giving each session its own conversation context across turns. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3 per turn.
- **Guardrail-driven retry**: when `ResponseGuardrail` rejects a candidate reply, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail, the workflow's `converseTurnStep` fails over to `error` and the entity records a `TurnFailed` event.
- **Summarizer is synchronous and deterministic**: `TurnSummarizer` runs in-process inside `summarizeStep`. No LLM call, no external service — the same turn list always produces the same summary score. This is the single-agent guarantee.
- **No saga / no compensation**: session state is append-only. There is nothing to roll back; a failed session preserves its partial turn history for operator review.
