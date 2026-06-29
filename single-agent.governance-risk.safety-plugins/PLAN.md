# PLAN — safety-plugins

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

  API[SafetyEndpoint]:::ep
  Entity[SafetyEntity]:::ese
  Sanitizer[PayloadSanitizer]:::cons
  WF[SafetyWorkflow]:::wf
  Agent[SafetyAgent]:::agent
  InGuard[InputGuardrail]:::guard
  OutGuard[OutputGuardrail]:::guard
  Evaluator[DecisionEvaluator]:::guard
  View[SafetyView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|PayloadSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|screenStep runSingleTask| Agent
  Agent -.->|before-llm-call| InGuard
  InGuard -.->|hard-block or pass| Agent
  Agent -.->|after-llm-response| OutGuard
  OutGuard -.->|accept or retry| Agent
  Agent -->|SafetyDecision| WF
  WF -->|recordDecision| Entity
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
  participant API as SafetyEndpoint
  participant E as SafetyEntity
  participant S as PayloadSanitizer
  participant W as SafetyWorkflow
  participant IG as InputGuardrail
  participant A as SafetyAgent
  participant OG as OutputGuardrail
  participant Ev as DecisionEvaluator

  U->>API: POST /api/screenings
  API->>E: submit(request)
  E-->>API: { screeningId }
  E-.->>S: PayloadSubmitted
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(screeningId)
  W->>E: poll getScreening
  E-->>W: sanitized.isPresent()
  W->>E: markScreening
  W->>IG: before-llm-call(payload)
  IG-->>W: pass
  W->>A: runSingleTask(rules + attachment)
  A->>OG: after-llm-response(candidate)
  OG-->>A: accept
  A-->>W: SafetyDecision
  W->>E: recordDecision(decision)
  W->>Ev: score(decision, rules)
  Ev-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `SafetyEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: PayloadSanitized
  SANITIZED --> SCREENING: ScreeningStarted
  SCREENING --> DECISION_RECORDED: DecisionRecorded
  DECISION_RECORDED --> EVALUATED: EvaluationScored
  SUBMITTED --> FAILED: ScreeningFailed (sanitizer error)
  SCREENING --> FAILED: ScreeningFailed (hard-block or agent error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SafetyEntity ||--o{ PayloadSubmitted : emits
  SafetyEntity ||--o{ PayloadSanitized : emits
  SafetyEntity ||--o{ ScreeningStarted : emits
  SafetyEntity ||--o{ DecisionRecorded : emits
  SafetyEntity ||--o{ EvaluationScored : emits
  SafetyEntity ||--o{ ScreeningFailed : emits
  SafetyView }o--|| SafetyEntity : projects
  PayloadSanitizer }o--|| SafetyEntity : subscribes
  SafetyWorkflow }o--|| SafetyEntity : reads-and-writes
  SafetyAgent ||--o{ SafetyDecision : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SafetyEndpoint` | `api/SafetyEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SafetyEntity` | `application/SafetyEntity.java` (state in `domain/Screening.java`, events in `domain/ScreeningEvent.java`) |
| `PayloadSanitizer` | `application/PayloadSanitizer.java` |
| `SafetyWorkflow` | `application/SafetyWorkflow.java` |
| `SafetyAgent` | `application/SafetyAgent.java` (tasks in `application/SafetyTasks.java`) |
| `InputGuardrail` | `application/InputGuardrail.java` |
| `OutputGuardrail` | `application/OutputGuardrail.java` |
| `DecisionEvaluator` | `application/DecisionEvaluator.java` |
| `SafetyView` | `application/SafetyView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `screenStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SafetyWorkflow::error)`. The 60 s on `screenStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"screening-" + screeningId` as the workflow id; the `PayloadSanitizer` Consumer is allowed to redeliver `PayloadSubmitted` events because `SafetyEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized screening is a no-op.
- **One agent per screening**: the AutonomousAgent instance id is `"screener-" + screeningId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **InputGuardrail hard-block path**: when `InputGuardrail` fires, the task terminates without an LLM call. The workflow's `screenStep` receives a rejection result and fails over to `error`, which transitions the entity to `FAILED`. The `ScreeningFailed` reason carries the matching injection-pattern id.
- **OutputGuardrail-driven retry**: when `OutputGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `screenStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `DecisionEvaluator` runs in-process inside `evalStep`. No LLM call, no external service — the same decision always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
