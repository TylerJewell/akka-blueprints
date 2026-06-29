# PLAN — document-forge-agent

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

  API[ForgeEndpoint]:::ep
  Entity[ForgeEntity]:::ese
  Consumer[ForgeOutputConsumer]:::cons
  WF[ForgeWorkflow]:::wf
  Agent[DocumentForgeAgent]:::agent
  Guard[WriteDocumentGuardrail]:::guard
  Auditor[ForgeAuditor]:::guard
  View[ForgeView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|forgeStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|validate path+content| Agent
  Agent -->|ForgeResult| WF
  WF -->|completeForge| Entity
  Entity -.->|ForgeCompleted| Consumer
  Consumer -->|attachAudit| Entity
  WF -->|auditStep score| Auditor
  Auditor -->|AuditEntry| WF
  WF -->|attachAudit| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ForgeEndpoint
  participant E as ForgeEntity
  participant W as ForgeWorkflow
  participant A as DocumentForgeAgent
  participant G as WriteDocumentGuardrail
  participant Au as ForgeAuditor
  participant C as ForgeOutputConsumer

  U->>API: POST /api/forges
  API->>E: submit(request)
  E-->>API: { forgeId }
  API->>W: start(forgeId)
  W->>E: markForging
  W->>A: runSingleTask(prompt + style template)
  A->>G: before-tool-call(write_document args)
  G-->>A: accept
  A-->>W: ForgeResult
  W->>E: completeForge(result)
  E-.->>C: ForgeCompleted
  C->>C: quality check
  C->>E: attachAudit
  W->>Au: score(result)
  Au-->>W: AuditEntry
  W->>E: attachAudit(audit)
  E-.->>U: SSE event(AUDITED)
```

## State machine — `ForgeEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> FORGING: ForgeStarted
  FORGING --> FORGE_COMPLETED: ForgeCompleted
  FORGE_COMPLETED --> AUDITED: ForgeAudited
  FORGING --> FAILED: ForgeFailed (agent error)
  SUBMITTED --> FAILED: ForgeFailed (workflow error)
  AUDITED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ForgeEntity ||--o{ ForgeSubmitted : emits
  ForgeEntity ||--o{ ForgeStarted : emits
  ForgeEntity ||--o{ ForgeCompleted : emits
  ForgeEntity ||--o{ ForgeAudited : emits
  ForgeEntity ||--o{ ForgeFailed : emits
  ForgeView }o--|| ForgeEntity : projects
  ForgeOutputConsumer }o--|| ForgeEntity : subscribes
  ForgeWorkflow }o--|| ForgeEntity : reads-and-writes
  DocumentForgeAgent ||--o{ ForgeResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ForgeEndpoint` | `api/ForgeEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ForgeEntity` | `application/ForgeEntity.java` (state in `domain/ForgeRun.java`, events in `domain/ForgeEvent.java`) |
| `ForgeOutputConsumer` | `application/ForgeOutputConsumer.java` |
| `ForgeWorkflow` | `application/ForgeWorkflow.java` |
| `DocumentForgeAgent` | `application/DocumentForgeAgent.java` (tasks in `application/ForgeTasks.java`) |
| `WriteDocumentGuardrail` | `application/WriteDocumentGuardrail.java` |
| `ForgeAuditor` | `application/ForgeAuditor.java` |
| `ForgeView` | `application/ForgeView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `forgeStep` 60 s, `auditStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ForgeWorkflow::error)`. The 60 s on `forgeStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"forge-" + forgeId` as the workflow id. `ForgeEntity.completeForge` is event-version-guarded — a second completion attempt against an already-completed forge is a no-op.
- **One agent per forge**: the AutonomousAgent instance id is `"forger-" + forgeId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `WriteDocumentGuardrail` rejects a tool call, the rejection is returned to the agent loop as a structured error. If all 3 iterations produce guardrail-blocked tool calls, `forgeStep` fails over to `error` and the entity transitions to `FAILED`.
- **Audit is synchronous and deterministic**: `ForgeAuditor` runs in-process inside `auditStep`. No LLM call — the same output always scores the same. This preserves the single-agent promise.
- **No saga / no compensation**: every step is either append-only event write or a single-task agent call. There is nothing external to roll back.
