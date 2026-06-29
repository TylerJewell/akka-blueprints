# PLAN — service-agent-omnichannel

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

  API[CaseEndpoint]:::ep
  Entity[CaseEntity]:::ese
  Sanitizer[MessageSanitizer]:::cons
  WF[CaseWorkflow]:::wf
  Agent[ServiceAgent]:::agent
  ReplyG[ReplyGuardrail]:::guard
  CrmG[CrmWriteGuardrail]:::guard
  Scorer[EscalationScorer]:::guard
  View[CaseView]:::view
  App[AppEndpoint]:::ep

  API -->|ingest| Entity
  Entity -.->|MessageReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|triageStep classify| Entity
  WF -->|handleStep runSingleTask| Agent
  Agent -.->|before-agent-response| ReplyG
  Agent -.->|before-tool-call| CrmG
  Agent -->|AgentReply| WF
  WF -->|recordReply| Entity
  WF -->|escalate if ESCALATE| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as CaseEndpoint
  participant E as CaseEntity
  participant S as MessageSanitizer
  participant W as CaseWorkflow
  participant A as ServiceAgent
  participant RG as ReplyGuardrail
  participant CG as CrmWriteGuardrail
  participant Sc as EscalationScorer

  U->>API: POST /api/cases
  API->>E: ingest(message)
  E-->>API: { caseId }
  E-.->>S: MessageReceived
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(caseId)
  W->>E: poll getCase
  E-->>W: sanitized.isPresent()
  W->>E: recordTriage(BILLING)
  W->>E: markHandling
  W->>A: runSingleTask(context + attachment)
  A->>CG: before-tool-call(case.create)
  CG-->>A: accept
  A->>RG: before-agent-response(candidate)
  RG-->>A: accept
  A-->>W: AgentReply
  W->>E: recordReply(reply)
  W->>Sc: score(reply, context)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(REPLIED)
```

## State machine — `CaseEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: MessageSanitized
  SANITIZED --> TRIAGING: CaseTriaged
  TRIAGING --> HANDLING: HandlingStarted
  HANDLING --> REPLIED: ReplySent
  REPLIED --> ESCALATED: CaseEscalated (resolutionIntent=ESCALATE)
  REPLIED --> RESOLVED: CaseResolved
  HANDLING --> FAILED: CaseFailed (agent error)
  RECEIVED --> FAILED: CaseFailed (sanitizer error)
  ESCALATED --> [*]
  RESOLVED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  CaseEntity ||--o{ MessageReceived : emits
  CaseEntity ||--o{ MessageSanitized : emits
  CaseEntity ||--o{ CaseTriaged : emits
  CaseEntity ||--o{ HandlingStarted : emits
  CaseEntity ||--o{ ReplySent : emits
  CaseEntity ||--o{ CaseEscalated : emits
  CaseEntity ||--o{ CaseResolved : emits
  CaseEntity ||--o{ EvaluationScored : emits
  CaseEntity ||--o{ CaseFailed : emits
  CaseView }o--|| CaseEntity : projects
  MessageSanitizer }o--|| CaseEntity : subscribes
  CaseWorkflow }o--|| CaseEntity : reads-and-writes
  ServiceAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CaseEndpoint` | `api/CaseEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CaseEntity` | `application/CaseEntity.java` (state in `domain/Case.java`, events in `domain/CaseEvent.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `CaseWorkflow` | `application/CaseWorkflow.java` |
| `ServiceAgent` | `application/ServiceAgent.java` (tasks in `application/CaseTasks.java`) |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `CrmWriteGuardrail` | `application/CrmWriteGuardrail.java` |
| `CaseTriage` | `application/CaseTriage.java` |
| `EscalationScorer` | `application/EscalationScorer.java` |
| `CaseView` | `application/CaseView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `triageStep` 5 s, `handleStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CaseWorkflow::error)`. The 60 s on `handleStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"case-" + caseId` as the workflow id; the `MessageSanitizer` Consumer is allowed to redeliver `MessageReceived` events because `CaseEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized case is a no-op.
- **One agent per case**: the AutonomousAgent instance id is `"agent-" + caseId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Two guardrails, one agent**: both `ReplyGuardrail` (before-agent-response) and `CrmWriteGuardrail` (before-tool-call) are registered on the same `ServiceAgent` definition. They run at different points in the agent loop — the tool-call guard fires before each tool invocation; the response guard fires before each proposed final reply. Both count toward `maxIterationsPerTask`.
- **Escalation is terminal**: once `CaseEntity.escalate` is called, the workflow's `handleStep` does not retry the agent. The case sits in `ESCALATED` state for human resolution. No saga compensation is required — CRM writes that landed before escalation are left as-is.
- **Triage is synchronous and deterministic**: `CaseTriage.classify` runs in-process inside `triageStep`. No LLM call.
- **Eval is synchronous and deterministic**: `EscalationScorer` runs in-process inside `evalStep`. No LLM call — the same reply for the same context always scores the same.
