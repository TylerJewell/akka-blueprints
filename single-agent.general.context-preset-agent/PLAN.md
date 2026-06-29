# PLAN — context-preset-agent

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef kve fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[PresetRequestEndpoint]:::ep
  Entity[PresetRequestEntity]:::ese
  Registry[PresetRegistry]:::kve
  WF[PresetRequestWorkflow]:::wf
  Agent[ContextPresetAgent]:::agent
  Guard[ToolGatingGuardrail]:::guard
  View[PresetRequestView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|resolvePresetStep getPreset| Registry
  Registry -->|PresetDefinition| WF
  WF -->|resolvePreset| Entity
  WF -->|executeStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|INVOKED or BLOCKED| Agent
  Agent -->|PresetRequestResult| WF
  WF -->|complete| Entity
  WF -->|auditStep recordAudit| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path: prod + admin)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as PresetRequestEndpoint
  participant E as PresetRequestEntity
  participant WF as PresetRequestWorkflow
  participant R as PresetRegistry
  participant A as ContextPresetAgent
  participant G as ToolGatingGuardrail

  U->>API: POST /api/preset-requests
  API->>E: submit(request)
  API->>WF: start(requestId)
  E-->>API: { requestId }
  WF->>R: getPreset("prod:admin")
  R-->>WF: PresetDefinition
  WF->>E: resolvePreset(preset)
  WF->>E: markExecuting
  WF->>A: runSingleTask(instructions + requestText)
  A->>G: before-tool-call(readStateTool)
  G-->>A: INVOKED
  A->>G: before-tool-call(adminActionTool)
  G-->>A: INVOKED (admin in allowedTools)
  A-->>WF: PresetRequestResult
  WF->>E: complete(result)
  WF->>E: recordAudit(auditEntry)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `PresetRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PRESET_RESOLVED: PresetResolved
  PRESET_RESOLVED --> EXECUTING: ExecutionStarted
  EXECUTING --> COMPLETED: RequestCompleted
  COMPLETED --> COMPLETED: RequestAudited
  EXECUTING --> FAILED: RequestFailed (agent error)
  SUBMITTED --> FAILED: RequestFailed (preset not found)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PresetRequestEntity ||--o{ RequestSubmitted : emits
  PresetRequestEntity ||--o{ PresetResolved : emits
  PresetRequestEntity ||--o{ ExecutionStarted : emits
  PresetRequestEntity ||--o{ RequestCompleted : emits
  PresetRequestEntity ||--o{ RequestAudited : emits
  PresetRequestEntity ||--o{ RequestFailed : emits
  PresetRequestView }o--|| PresetRequestEntity : projects
  PresetRequestWorkflow }o--|| PresetRequestEntity : reads-and-writes
  PresetRequestWorkflow }o--|| PresetRegistry : reads
  ContextPresetAgent ||--o{ PresetRequestResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PresetRequestEndpoint` | `api/PresetRequestEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PresetRequestEntity` | `application/PresetRequestEntity.java` (state in `domain/PresetRequestState.java`, events in `domain/PresetRequestEvent.java`) |
| `PresetRegistry` | `application/PresetRegistry.java` |
| `PresetRequestWorkflow` | `application/PresetRequestWorkflow.java` |
| `ContextPresetAgent` | `application/ContextPresetAgent.java` (tasks in `application/PresetRequestTasks.java`) |
| `ToolGatingGuardrail` | `application/ToolGatingGuardrail.java` |
| `PresetRequestView` | `application/PresetRequestView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `resolvePresetStep` 5 s, `executeStep` 60 s, `auditStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(PresetRequestWorkflow::error)`. The 60 s on `executeStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"preset-req-" + requestId` as the workflow id; `PresetRequestEntity.resolvePreset` is event-version-guarded — a second resolve attempt against an already-resolved request is a no-op.
- **One agent per request**: the AutonomousAgent instance id is `"agent-" + requestId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` bounds the call.
- **Guardrail-driven blocking**: when `ToolGatingGuardrail` blocks a tool call, the rejection is returned as a structured permission-denied error to the agent loop. The agent does not retry the blocked call — it explains the restriction and proceeds to completion. The blocked entry is recorded in `toolCallLog` with `status = BLOCKED`. This is different from a before-agent-response retry: the task still completes; it just completes with blocked-tool evidence in the log.
- **Audit is synchronous and deterministic**: `auditStep` counts INVOKED vs BLOCKED entries in the completed result's `toolCallLog` — no LLM call, no external service. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either a pure Key-Value read, append-only event write, or a single-task agent call. Nothing external to roll back.
