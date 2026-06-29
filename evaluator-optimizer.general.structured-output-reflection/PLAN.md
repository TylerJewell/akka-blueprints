# PLAN — structured-output-reflection

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Gen[GeneratorAgent]:::agent
  Critic[CriticAgent]:::agent

  WF[ReflectionWorkflow]:::wf
  GenEnt[GenerationEntity]:::ese
  Queue[SubmissionQueue]:::ese
  View[GenerationsView]:::view
  Consumer[SubmissionConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[GenerationEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue request| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Gen
  WF -->|validate| Critic
  WF -->|emit events| GenEnt
  GenEnt -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| GenEnt
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as GenerationEndpoint
  participant Q as SubmissionQueue
  participant SC as SubmissionConsumer
  participant W as ReflectionWorkflow
  participant G as GeneratorAgent
  participant C as CriticAgent
  participant E as GenerationEntity
  participant V as GenerationsView

  U->>API: POST /api/generations {schemaName, prompt}
  API->>Q: append RequestSubmitted
  API-->>U: 202 {generationId}
  Q->>SC: RequestSubmitted
  SC->>W: start({generationId, schemaName, prompt, maxAttempts=4})
  W->>E: emit GenerationCreated (GENERATING)

  W->>G: GENERATE(schemaName, prompt)
  G-->>W: GeneratedOutput #1 (parseable=true)
  W->>E: emit AttemptGenerated (n=1)
  Note over W: guardrailStep (deterministic schema check)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>E: status VALIDATING
  W->>C: VALIDATE(GeneratedOutput #1)
  C-->>W: ValidationReport{REVISE, score=3, 3 bullets}
  W->>E: emit AttemptValidated (n=1, REVISE)

  W->>G: REVISE_OUTPUT(schemaName, prompt, prior, notes)
  G-->>W: GeneratedOutput #2 (parseable=true)
  W->>E: emit AttemptGenerated (n=2)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>C: VALIDATE(GeneratedOutput #2)
  C-->>W: ValidationReport{PASS, score=5, rationale}
  W->>E: emit AttemptValidated (n=2, PASS)
  W->>E: emit GenerationPassed (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `GenerationEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> VALIDATING: AttemptGenerated + guardrail passed
  GENERATING --> GENERATING: guardrail blocked, re-generate
  VALIDATING --> GENERATING: ValidationReport = REVISE, attempts < max
  VALIDATING --> PASSED: ValidationReport = PASS
  VALIDATING --> FAILED_FINAL: ValidationReport = REVISE, attempts = max
  PASSED --> [*]
  FAILED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  GenerationEntity ||--o{ GenerationCreated : emits
  GenerationEntity ||--o{ AttemptGenerated : emits
  GenerationEntity ||--o{ AttemptGuardrailVerdictRecorded : emits
  GenerationEntity ||--o{ AttemptValidated : emits
  GenerationEntity ||--o{ GenerationPassed : emits
  GenerationEntity ||--o{ GenerationFailedFinal : emits
  GenerationEntity ||--o{ EvalRecorded : emits
  GenerationsView }o--|| GenerationEntity : projects
  SubmissionQueue ||--o{ RequestSubmitted : emits
  SubmissionConsumer }o--|| SubmissionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `CriticAgent` | `application/CriticAgent.java` |
| `GenerationTasks` | `application/GenerationTasks.java` |
| `ReflectionWorkflow` | `application/ReflectionWorkflow.java` |
| `GenerationEntity` | `application/GenerationEntity.java` (state in `domain/Generation.java`, events in `domain/GenerationEvent.java`) |
| `SubmissionQueue` | `application/SubmissionQueue.java` |
| `GenerationsView` | `application/GenerationsView.java` |
| `SubmissionConsumer` | `application/SubmissionConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `GenerationEndpoint` | `api/GenerationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `validateStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — the workflow degrades to `FAILED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `GenerationEndpoint.submit` uses `(schemaName, prompt, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(generationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `structured-output-reflection.max-attempts` (default 4). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the best output and every validation report on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it attempts JSON parsing and verifies required fields for the named schema against a static registry. On failure it returns a structured `ValidationNotes` payload with a single diagnostic bullet. The diagnostic is deterministic and never becomes an LLM-generated critique.
