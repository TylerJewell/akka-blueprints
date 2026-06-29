# PLAN — mcp-github-bot

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
  classDef halt fill:#1a1a2e,stroke:#e94560,color:#e94560;

  API[BotSessionEndpoint]:::ep
  App[AppEndpoint]:::ep
  SessionEntity[BotSessionEntity]:::ese
  HaltEntity[HaltFlagEntity]:::halt
  WF[BotSessionWorkflow]:::wf
  Agent[GitHubBotAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  View[BotSessionView]:::view

  API -->|submit| SessionEntity
  API -->|enableHalt / disableHalt| HaltEntity
  API -->|start workflow| WF
  WF -->|runAgentStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|reads flag| HaltEntity
  Guard -->|allow / reject| Agent
  Agent -->|BotResponse| WF
  WF -->|complete / fail| SessionEntity
  SessionEntity -.->|projects| View
  API -->|list / SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, no halt)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as BotSessionEndpoint
  participant SE as BotSessionEntity
  participant HE as HaltFlagEntity
  participant WF as BotSessionWorkflow
  participant A as GitHubBotAgent
  participant G as ToolCallGuardrail
  participant GH as GitHub MCP Server

  U->>API: POST /api/sessions
  API->>SE: submit(request)
  SE-->>API: { sessionId }
  API->>WF: start(sessionId, githubToken)
  WF->>SE: start()
  WF->>A: runSingleTask(instructions)
  A->>G: before-tool-call(list_issues)
  G->>HE: getFlag()
  HE-->>G: writeHalted=false
  G-->>A: allow
  A->>GH: list_issues(owner/repo)
  GH-->>A: [ issue list ]
  A-->>WF: BotResponse
  WF->>SE: complete(response)
  SE-.->>U: SSE event(COMPLETED)
```

## State machine — `BotSessionEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RUNNING: SessionStarted
  RUNNING --> COMPLETED: SessionCompleted
  RUNNING --> FAILED: SessionFailed (agent error / halt)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BotSessionEntity ||--o{ SessionSubmitted : emits
  BotSessionEntity ||--o{ SessionStarted : emits
  BotSessionEntity ||--o{ SessionCompleted : emits
  BotSessionEntity ||--o{ SessionFailed : emits
  HaltFlagEntity ||--o{ WriteHaltEnabled : emits
  HaltFlagEntity ||--o{ WriteHaltDisabled : emits
  BotSessionView }o--|| BotSessionEntity : projects
  BotSessionWorkflow }o--|| BotSessionEntity : reads-and-writes
  BotSessionWorkflow }o--|| GitHubBotAgent : invokes
  ToolCallGuardrail }o--|| HaltFlagEntity : reads
  GitHubBotAgent ||--o{ BotResponse : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BotSessionEndpoint` | `api/BotSessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BotSessionEntity` | `application/BotSessionEntity.java` (state in `domain/BotSession.java`, events in `domain/BotSessionEvent.java`) |
| `HaltFlagEntity` | `application/HaltFlagEntity.java` (state in `domain/HaltFlag.java`, events in `domain/HaltFlagEvent.java`) |
| `BotSessionWorkflow` | `application/BotSessionWorkflow.java` |
| `GitHubBotAgent` | `application/GitHubBotAgent.java` (tasks in `application/BotTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `BotSessionView` | `application/BotSessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| `MockGitHubMcpServer` (option-a only) | `application/MockGitHubMcpServer.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `runAgentStep` 90 s (accommodates multiple MCP round-trips), `recordStep` 10 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(BotSessionWorkflow::error)`. The 90 s covers multi-tool chains (Lesson 4).
- **Idempotency**: workflow id is `"session-" + sessionId`; `BotSessionEntity.start` is event-version-guarded — a second start attempt on an already-running session is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"bot-" + sessionId`, giving each task its own tool-call context. The agent's `capability(...).maxIterationsPerTask(5)` caps the tool-call loop.
- **Guardrail on every write tool call**: `ToolCallGuardrail` reads `HaltFlagEntity` synchronously on every write-class dispatch. The read is a local componentClient call (no network hop outside the runtime). Flag changes take effect on the next tool call — there is no TTL or cache.
- **Token isolation**: `BotRequest.githubToken` is passed to `BotSessionWorkflow` as a workflow-start parameter; it is forwarded to the `McpServerConfig` at agent invocation time. The field is OMITTED from `SessionSubmitted` event and from `BotSession` entity state. It never appears in the view, in any response body, or in logs.
- **No saga / no compensation**: each workflow step is either an agent call or a single entity write. There is no distributed transaction to roll back.
