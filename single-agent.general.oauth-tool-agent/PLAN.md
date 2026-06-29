# PLAN — ae-oauth

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
  classDef reg fill:#1a1a0e,stroke:#d4a017,color:#d4a017;

  API[SessionEndpoint]:::ep
  Entity[SessionEntity]:::ese
  WF[SessionWorkflow]:::wf
  Agent[ToolCallerAgent]:::agent
  Guard[OAuthScopeGuardrail]:::guard
  TokenReg[TokenRegistry]:::reg
  ToolReg[ToolRegistry]:::reg
  Executor[SimulatedToolExecutor]:::reg
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start workflow| WF
  WF -->|resolveTokenStep| TokenReg
  TokenReg -->|TokenSpec| WF
  WF -->|TokenResolved| Entity
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|lookup| ToolReg
  Guard -->|allow/deny| Agent
  Agent -->|SimulatedToolExecutor| Executor
  Agent -->|AgentResult| WF
  WF -->|recordResult| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path: full-access token)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SessionEndpoint
  participant E as SessionEntity
  participant W as SessionWorkflow
  participant TR as TokenRegistry
  participant A as ToolCallerAgent
  participant G as OAuthScopeGuardrail
  participant Ex as SimulatedToolExecutor

  U->>API: POST /api/sessions
  API->>E: create(request)
  E-->>API: { sessionId }
  API->>W: start(sessionId)
  W->>TR: resolve(tokenId)
  TR-->>W: TokenSpec{scopes:[calendar:read,...]}
  W->>E: resolveToken(token)
  W->>E: markRunning
  W->>A: runSingleTask(request + scopes)
  A->>G: before-tool-call(listCalendarEvents)
  G-->>A: allow (calendar:read present)
  A->>Ex: listCalendarEvents()
  Ex-->>A: [{id:evt-1,...},...]
  A->>G: before-tool-call(createCalendarEvent)
  G-->>A: allow (calendar:write present)
  A->>Ex: createCalendarEvent(...)
  Ex-->>A: {id:evt-new-42}
  A-->>W: AgentResult{outcome:SUCCESS,...}
  W->>E: recordResult(result)
  W->>E: complete()
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> TOKEN_RESOLVED: TokenResolved
  TOKEN_RESOLVED --> RUNNING: AgentStarted
  RUNNING --> COMPLETED: ResultRecorded
  PENDING --> FAILED: SessionFailed (expired token)
  RUNNING --> FAILED: SessionFailed (agent error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ TokenResolved : emits
  SessionEntity ||--o{ AgentStarted : emits
  SessionEntity ||--o{ ResultRecorded : emits
  SessionEntity ||--o{ SessionFailed : emits
  SessionView }o--|| SessionEntity : projects
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  ToolCallerAgent ||--o{ AgentResult : returns
  OAuthScopeGuardrail }o--|| ToolRegistry : consults
  SessionWorkflow }o--|| TokenRegistry : resolves
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `ToolCallerAgent` | `application/ToolCallerAgent.java` (tasks in `application/SessionTasks.java`) |
| `OAuthScopeGuardrail` | `application/OAuthScopeGuardrail.java` |
| `TokenRegistry` | `application/TokenRegistry.java` |
| `ToolRegistry` | `application/ToolRegistry.java` |
| `SimulatedToolExecutor` | `application/SimulatedToolExecutor.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `resolveTokenStep` 5 s, `agentStep` 90 s, `recordStep` 5 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(SessionWorkflow::error)`. The 90 s on `agentStep` accommodates multi-tool LLM round trips (Lesson 4).
- **Idempotency**: every workflow uses `"session-" + sessionId` as the workflow id. `SessionEntity.resolveToken` is version-guarded — a second `TokenResolved` event against an already-resolved session is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"agent-" + sessionId`, giving each task its own conversation context. `capability(...).maxIterationsPerTask(5)` allows multiple tool-call rounds without re-starting the task.
- **Guardrail is per-call, not per-task**: `OAuthScopeGuardrail` fires on each individual proposed tool call inside the task, not once at the start. The agent may receive a mix of allow and deny responses in a single task execution, producing `outcome: PARTIAL`.
- **Expired-token fast-fail**: token expiry is checked in `resolveTokenStep`, before `ToolCallerAgent` is ever invoked. This avoids an LLM call on invalid credentials and produces a `FAILED` session with a clear reason.
- **No external API calls**: `SimulatedToolExecutor` returns deterministic in-process responses. The guardrail enforces real scope logic against `ToolRegistry`; the executor provides realistic-looking output without network dependency.
