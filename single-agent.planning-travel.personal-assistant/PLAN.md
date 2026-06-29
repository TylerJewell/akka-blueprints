# PLAN — personal-assistant

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

  API[AssistantEndpoint]:::ep
  Entity[AssistantEntity]:::ese
  Sanitizer[ContextSanitizer]:::cons
  WF[AssistantWorkflow]:::wf
  Agent[AssistantAgent]:::agent
  Guard[WriteGuardrail]:::guard
  Scorer[ActionScorer]:::guard
  View[AssistantView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|RequestSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|actStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|AssistantAction| WF
  WF -->|recordAction| Entity
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
  participant API as AssistantEndpoint
  participant E as AssistantEntity
  participant S as ContextSanitizer
  participant W as AssistantWorkflow
  participant A as AssistantAgent
  participant G as WriteGuardrail
  participant Sc as ActionScorer

  U->>API: POST /api/requests
  API->>E: submit(request)
  E-->>API: { requestId }
  E-.->>S: RequestSubmitted
  S->>S: redact PII from context
  S->>E: attachSanitized
  S->>W: start(requestId)
  W->>E: poll getState
  E-->>W: sanitized.isPresent()
  W->>E: markActing
  W->>A: runSingleTask(request text + context attachment)
  A->>G: before-tool-call(proposed write)
  G-->>A: accept
  A-->>W: AssistantAction
  W->>E: recordAction(action)
  W->>Sc: score(action, request)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `AssistantEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: ContextSanitized
  SANITIZED --> ACTING: ActionStarted
  ACTING --> ACTION_RECORDED: ActionRecorded
  ACTION_RECORDED --> EVALUATED: EvaluationScored
  ACTING --> FAILED: RequestFailed (agent error)
  SUBMITTED --> FAILED: RequestFailed (sanitizer error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  AssistantEntity ||--o{ RequestSubmitted : emits
  AssistantEntity ||--o{ ContextSanitized : emits
  AssistantEntity ||--o{ ActionStarted : emits
  AssistantEntity ||--o{ ActionRecorded : emits
  AssistantEntity ||--o{ EvaluationScored : emits
  AssistantEntity ||--o{ RequestFailed : emits
  AssistantView }o--|| AssistantEntity : projects
  ContextSanitizer }o--|| AssistantEntity : subscribes
  AssistantWorkflow }o--|| AssistantEntity : reads-and-writes
  AssistantAgent ||--o{ AssistantAction : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AssistantEndpoint` | `api/AssistantEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AssistantEntity` | `application/AssistantEntity.java` (state in `domain/AssistantState.java`, events in `domain/AssistantEvent.java`) |
| `ContextSanitizer` | `application/ContextSanitizer.java` |
| `AssistantWorkflow` | `application/AssistantWorkflow.java` |
| `AssistantAgent` | `application/AssistantAgent.java` (tasks in `application/AssistantTasks.java`) |
| `WriteGuardrail` | `application/WriteGuardrail.java` |
| `ActionScorer` | `application/ActionScorer.java` |
| `AssistantView` | `application/AssistantView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `actStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AssistantWorkflow::error)`. The 60 s on `actStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"assistant-" + requestId` as the workflow id; the `ContextSanitizer` Consumer is allowed to redeliver `RequestSubmitted` events because `AssistantEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized request is a no-op.
- **One agent per request**: the AutonomousAgent instance id is `"assistant-" + requestId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `WriteGuardrail` rejects a proposed tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `actStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `ActionScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same action always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
