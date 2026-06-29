# PLAN — managed-harness-tool-use-agent

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
  classDef support fill:#1a1a1a,stroke:#888,color:#888;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[QueryWorkflow]:::wf
  Agent[OperationsAgent]:::agent
  Guard[ToolAllowlistGuardrail]:::guard
  Stubs[ToolStubs]:::support
  Profiles[AgentProfiles]:::support
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit + start workflow| Entity
  API -->|start| WF
  WF -->|markRunning| Entity
  WF -->|runStep runSingleTask| Agent
  Agent -.->|before-tool-invocation| Guard
  Guard -.->|allow / reject| Agent
  Agent -->|tool dispatch| Stubs
  Stubs -->|tool output| Agent
  Agent -->|QueryAnswer| WF
  WF -->|recordStep recordAnswer| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  WF -->|fail on error| Entity
  Profiles -.->|allowlist lookup| Guard
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant W as QueryWorkflow
  participant A as OperationsAgent
  participant G as ToolAllowlistGuardrail
  participant T as ToolStubs

  U->>API: POST /api/queries
  API->>E: submit(request)
  API->>W: start(queryId)
  E-->>API: { queryId }
  W->>E: markRunning
  W->>A: runSingleTask(question + profile)
  A->>G: before-tool-invocation(query_metrics)
  G-->>A: allow
  A->>T: query_metrics(namespace, metric, window)
  T-->>A: metric data
  A->>G: before-tool-invocation(fetch_logs)
  G-->>A: allow
  A->>T: fetch_logs(service, ERROR, 100)
  T-->>A: log entries
  A-->>W: QueryAnswer(prose, toolTrace)
  W->>E: recordAnswer(answer)
  E-.->>U: SSE event(ANSWERED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RUNNING: AgentStarted
  RUNNING --> ANSWERED: AnswerRecorded
  RUNNING --> FAILED: QueryFailed (agent error or guardrail-exhaustion)
  SUBMITTED --> FAILED: QueryFailed (workflow error)
  ANSWERED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ AgentStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  OperationsAgent ||--o{ QueryAnswer : returns
  OperationsAgent }o--|| ToolAllowlistGuardrail : checked-by
  OperationsAgent }o--|| ToolStubs : dispatches-to
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `OperationsAgent` | `application/OperationsAgent.java` (tasks in `application/QueryTasks.java`) |
| `ToolAllowlistGuardrail` | `application/ToolAllowlistGuardrail.java` |
| `AgentProfiles` | `application/AgentProfiles.java` |
| `ToolStubs` | `application/ToolStubs.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `runStep` 90 s, `recordStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 90 s on `runStep` accommodates multi-turn tool-calling latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; duplicate `POST /api/queries` calls with the same `queryId` resolve to no-ops at the entity level because `QuerySubmitted` is version-guarded.
- **One agent per query**: the AutonomousAgent instance id is `"ops-" + queryId`, giving each task its own tool-calling conversation context. `maxIterationsPerTask(8)` caps the tool-calling loop at eight turns.
- **Guardrail-driven recovery**: when `ToolAllowlistGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; the agent can attempt a different, allowed tool. If all 8 iterations are exhausted without a final answer, the workflow's `runStep` fails over to `error` and the entity transitions to `FAILED`.
- **Tool stubs are synchronous and deterministic**: `ToolStubs` methods are pure functions. No external service, no I/O. The same arguments always produce the same output. This keeps local-dev latency predictable and the single-agent invariant honest.
- **No saga / no compensation**: every step is either an append-only entity write or a single-task agent call. There is nothing external to roll back.
