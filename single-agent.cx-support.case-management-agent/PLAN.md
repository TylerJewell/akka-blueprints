# PLAN — case-management-agent

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
  Agent[SupportAgent]:::agent
  AG[ActionGuardrail]:::guard
  TG[ToolCallGuardrail]:::guard
  Evaluator[ActionEvaluator]:::guard
  View[CaseView]:::view
  App[AppEndpoint]:::ep

  API -->|receiveMessage| Entity
  Entity -.->|MessageReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-agent-response| AG
  AG -.->|pass / reject| Agent
  Agent -.->|before-tool-call| TG
  TG -.->|pass / reject| Agent
  Agent -->|CaseAction| WF
  WF -->|applyAction| Entity
  WF -->|evalStep score| Evaluator
  Evaluator -->|EvalResult| WF
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
  participant A as SupportAgent
  participant AG as ActionGuardrail
  participant TG as ToolCallGuardrail
  participant Ev as ActionEvaluator

  U->>API: POST /api/messages
  API->>E: receiveMessage(inbound)
  E-->>API: { messageId, caseId }
  E-.->>S: MessageReceived
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(messageId)
  W->>E: poll getCase
  E-->>W: sanitizedMessage.isPresent()
  W->>E: markActing
  W->>A: runSingleTask(message context)
  A->>AG: before-agent-response(candidate)
  AG-->>A: accept
  A->>TG: before-tool-call(crm-create)
  TG-->>A: accept
  A-->>W: CaseAction
  W->>E: applyAction(action, crmRecord)
  W->>Ev: score(action, message)
  Ev-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(OPEN/IN_PROGRESS)
```

## State machine — `CaseEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> IN_PROGRESS: ActionApplied (CREATE / UPDATE)
  IN_PROGRESS --> ESCALATED: ActionApplied (ESCALATE)
  IN_PROGRESS --> RESOLVED: ActionApplied (CLOSE)
  ESCALATED --> IN_PROGRESS: ActionApplied (UPDATE)
  ESCALATED --> RESOLVED: ActionApplied (CLOSE)
  IN_PROGRESS --> FAILED: CaseFailed (agent error)
  OPEN --> FAILED: CaseFailed (sanitizer error)
  RESOLVED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  CaseEntity ||--o{ MessageReceived : emits
  CaseEntity ||--o{ MessageSanitized : emits
  CaseEntity ||--o{ AgentActing : emits
  CaseEntity ||--o{ ActionApplied : emits
  CaseEntity ||--o{ EvaluationScored : emits
  CaseEntity ||--o{ CaseFailed : emits
  CaseView }o--|| CaseEntity : projects
  MessageSanitizer }o--|| CaseEntity : subscribes
  CaseWorkflow }o--|| CaseEntity : reads-and-writes
  SupportAgent ||--o{ CaseAction : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CaseEndpoint` | `api/CaseEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CaseEntity` | `application/CaseEntity.java` (state in `domain/CaseRecord.java`, events in `domain/CaseEvent.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `CaseWorkflow` | `application/CaseWorkflow.java` |
| `SupportAgent` | `application/SupportAgent.java` (tasks in `application/CaseTasks.java`) |
| `ActionGuardrail` | `application/ActionGuardrail.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `ActionEvaluator` | `application/ActionEvaluator.java` |
| `CaseView` | `application/CaseView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `agentStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CaseWorkflow::error)`. The 60 s on `agentStep` accommodates LLM latency plus potential guardrail retries (Lesson 4).
- **Idempotency**: every workflow uses `"case-" + messageId` as its workflow id; `MessageSanitizer` is allowed to redeliver `MessageReceived` events because `CaseEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized case is a no-op.
- **One agent per message**: the AutonomousAgent instance id is `"agent-" + messageId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Two independent guardrails**: `ActionGuardrail` fires on every candidate response before the agent loop accepts it; `ToolCallGuardrail` fires before each tool invocation. They enforce distinct risk surfaces — structural correctness vs. CRM write policy — and neither silently covers the other.
- **Eval is synchronous and deterministic**: `ActionEvaluator` runs in-process inside `evalStep`. No LLM call, no external service — the same action always scores the same.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
