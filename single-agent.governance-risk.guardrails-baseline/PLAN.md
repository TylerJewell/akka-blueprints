# PLAN — guardrails-baseline

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

  API[ModerationEndpoint]:::ep
  Entity[ModerationEntity]:::ese
  Sanitizer[MessageSanitizer]:::cons
  WF[ModerationWorkflow]:::wf
  Agent[ModerationAgent]:::agent
  GIn[InputGuardrail]:::guard
  GOut[OutputGuardrail]:::guard
  Scorer[AuditScorer]:::guard
  View[ModerationView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|MessageSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|moderateStep runSingleTask| Agent
  Agent -.->|before-agent-invocation| GIn
  GIn -.->|tool-policy check| Agent
  Agent -.->|after-llm-response| GOut
  GOut -.->|structure check| Agent
  Agent -->|ModerationDecision| WF
  WF -->|recordDecision| Entity
  WF -->|auditStep score| Scorer
  Scorer -->|AuditResult| WF
  WF -->|recordAudit| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ModerationEndpoint
  participant E as ModerationEntity
  participant S as MessageSanitizer
  participant W as ModerationWorkflow
  participant A as ModerationAgent
  participant GIn as InputGuardrail
  participant GOut as OutputGuardrail
  participant Sc as AuditScorer

  U->>API: POST /api/moderations
  API->>E: submit(request)
  E-->>API: { moderationId }
  E-.->>S: MessageSubmitted
  S->>S: strip PII
  S->>E: attachSanitized
  S->>W: start(moderationId)
  W->>E: poll getModeration
  E-->>W: sanitized.isPresent()
  W->>E: markModerating
  W->>A: runSingleTask(rules + attachment)
  A->>GIn: before-agent-invocation(tool-list)
  GIn-->>A: accept
  A->>GOut: after-llm-response(candidate)
  GOut-->>A: accept
  A-->>W: ModerationDecision
  W->>E: recordDecision(decision)
  W->>Sc: score(decision, rules)
  Sc-->>W: AuditResult
  W->>E: recordAudit(audit)
  E-.->>U: SSE event(AUDITED)
```

## State machine — `ModerationEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: MessageSanitized
  SANITIZED --> MODERATING: ModerationStarted
  MODERATING --> DECISION_RECORDED: DecisionRecorded
  DECISION_RECORDED --> AUDITED: AuditCompleted
  MODERATING --> FAILED: ModerationFailed (agent or guardrail exhaustion)
  SUBMITTED --> FAILED: ModerationFailed (sanitizer error)
  AUDITED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ModerationEntity ||--o{ MessageSubmitted : emits
  ModerationEntity ||--o{ MessageSanitized : emits
  ModerationEntity ||--o{ ModerationStarted : emits
  ModerationEntity ||--o{ DecisionRecorded : emits
  ModerationEntity ||--o{ AuditCompleted : emits
  ModerationEntity ||--o{ ModerationFailed : emits
  ModerationView }o--|| ModerationEntity : projects
  MessageSanitizer }o--|| ModerationEntity : subscribes
  ModerationWorkflow }o--|| ModerationEntity : reads-and-writes
  ModerationAgent ||--o{ ModerationDecision : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ModerationEndpoint` | `api/ModerationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ModerationEntity` | `application/ModerationEntity.java` (state in `domain/Moderation.java`, events in `domain/ModerationEvent.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `ModerationWorkflow` | `application/ModerationWorkflow.java` |
| `ModerationAgent` | `application/ModerationAgent.java` (tasks in `application/ModerationTasks.java`) |
| `InputGuardrail` | `application/InputGuardrail.java` |
| `OutputGuardrail` | `application/OutputGuardrail.java` |
| `AuditScorer` | `application/AuditScorer.java` |
| `ModerationView` | `application/ModerationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `moderateStep` 60 s, `auditStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ModerationWorkflow::error)`. The 60 s on `moderateStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"moderation-" + moderationId` as the workflow id; `MessageSanitizer` Consumer redelivery is safe because `ModerationEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized moderation is a no-op.
- **One agent per moderation**: the AutonomousAgent instance id is `"moderator-" + moderationId`, giving each task its own conversation context. `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Two guardrails, one loop**: both `InputGuardrail` and `OutputGuardrail` are registered on `ModerationAgent`. The input guardrail fires once per task (before the first LLM call); the output guardrail fires once per LLM response. Both share the same iteration budget.
- **Audit is synchronous and deterministic**: `AuditScorer` runs in-process inside `auditStep`. No LLM call — the same decision always scores the same.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
