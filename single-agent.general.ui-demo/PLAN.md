# PLAN — ui-demo

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

  API[CopilotEndpoint]:::ep
  Entity[SessionEntity]:::ese
  WF[SessionWorkflow]:::wf
  Agent[CopilotAgent]:::agent
  Guard[ResponseGuardrail]:::guard
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|createSession / submitTurn| Entity
  API -->|start workflow| WF
  WF -->|agentCallStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|accept / reject| Agent
  Agent -->|CopilotResponse| WF
  WF -->|completeTurn| Entity
  WF -->|markIdle| Entity
  Entity -.->|projects| View
  API -->|list / SSE stream| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as CopilotEndpoint
  participant E as SessionEntity
  participant W as SessionWorkflow
  participant A as CopilotAgent
  participant G as ResponseGuardrail

  U->>API: POST /api/sessions/{id}/turns
  API->>E: submitTurn(turnId, prompt)
  API->>W: start(sessionId, turnId)
  W->>E: markStreaming(turnId)
  API-->>U: SSE stream open (TOKEN_CHUNK events begin)
  W->>A: runSingleTask(prompt + history context)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: CopilotResponse
  W->>E: completeTurn(turnId, response)
  E-.->>U: SSE RESPONSE_COMPLETE envelope
  W->>E: markIdle
```

## State machine — `SessionEntity` (per turn)

```mermaid
stateDiagram-v2
  [*] --> ACTIVE : SessionCreated
  ACTIVE --> ACTIVE : TurnSubmitted
  ACTIVE --> ACTIVE : StreamingStarted
  ACTIVE --> IDLE : TurnCompleted
  IDLE --> ACTIVE : TurnSubmitted (next turn)
  ACTIVE --> FAILED : TurnFailed (agent error)
  IDLE --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ TurnSubmitted : emits
  SessionEntity ||--o{ StreamingStarted : emits
  SessionEntity ||--o{ TurnCompleted : emits
  SessionEntity ||--o{ TurnFailed : emits
  SessionView }o--|| SessionEntity : projects
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  CopilotAgent ||--o{ CopilotResponse : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CopilotEndpoint` | `api/CopilotEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `CopilotAgent` | `application/CopilotAgent.java` (tasks in `application/CopilotTasks.java`) |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `agentCallStep` 60 s, `commitStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SessionWorkflow::error)`. The 60 s on `agentCallStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"session-" + sessionId + "-" + turnId` as the workflow id; submitting the same `turnId` twice is guarded by `SessionEntity.submitTurn`'s event-version check — a duplicate is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"copilot-" + sessionId`, which gives the conversation its own context window. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ResponseGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `agentCallStep` fails over to `error` and the turn transitions to `TURN_FAILED`.
- **Streaming**: the SSE stream from `GET /api/sessions/{id}/stream` begins immediately after `submitTurn`; token envelopes are forwarded from the agent task's streaming output. The `RESPONSE_COMPLETE` envelope is emitted only after `TurnCompleted` lands in the entity.
- **No saga / no compensation**: every step is either an append-only entity write or a single-task agent call. There is nothing external to roll back.
