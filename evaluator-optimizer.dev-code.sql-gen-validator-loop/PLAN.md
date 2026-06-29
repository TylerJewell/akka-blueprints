# PLAN — sql-gen-validator-loop

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
  Val[ValidatorAgent]:::agent

  WF[SqlWorkflow]:::wf
  QRE[QueryRequestEntity]:::ese
  Queue[RequestQueue]:::ese
  View[QueryRequestView]:::view
  Consumer[SubmissionConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[SqlEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue question| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Gen
  WF -->|validate| Val
  WF -->|emit events| QRE
  QRE -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| QRE
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SqlEndpoint
  participant Q as RequestQueue
  participant C as SubmissionConsumer
  participant W as SqlWorkflow
  participant G as GeneratorAgent
  participant V as ValidatorAgent
  participant E as QueryRequestEntity
  participant Vw as QueryRequestView

  U->>API: POST /api/queries {question, schemaName}
  API->>Q: append QuestionSubmitted
  API-->>U: 202 {requestId}
  Q->>C: QuestionSubmitted
  C->>W: start({requestId, question, schemaName, maxAttempts=4})
  W->>E: emit QueryRequestCreated (GENERATING)

  W->>G: GENERATE(question, schemaName)
  G-->>W: GeneratedQuery #1 (SELECT ...)
  W->>E: emit AttemptGenerated (n=1)
  Note over W: guardrailStep (deterministic keyword scan)
  W->>E: emit AttemptMutationGuardrailVerdictRecorded (passed=true)
  W->>E: status VALIDATING
  W->>V: VALIDATE(GeneratedQuery #1, schemaName)
  V-->>W: ValidationResult{INVALID, score=2, 3 bullets}
  W->>E: emit AttemptValidated (n=1, INVALID)

  W->>G: REVISE_QUERY(question, schemaName, prior, notes)
  G-->>W: GeneratedQuery #2 (revised SELECT ...)
  W->>E: emit AttemptGenerated (n=2)
  W->>E: emit AttemptMutationGuardrailVerdictRecorded (passed=true)
  W->>V: VALIDATE(GeneratedQuery #2, schemaName)
  V-->>W: ValidationResult{VALID, score=5, rationale}
  W->>E: emit AttemptValidated (n=2, VALID)
  W->>E: emit QueryRequestAccepted (n=2)
  E-->>Vw: project
  Vw-->>U: SSE update
```

## State machine — `QueryRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> VALIDATING: AttemptGenerated + guardrail passed
  GENERATING --> GENERATING: guardrail blocked, re-generate
  VALIDATING --> GENERATING: ValidationResult = INVALID, attempts < max
  VALIDATING --> ACCEPTED: ValidationResult = VALID
  VALIDATING --> FAILED_FINAL: ValidationResult = INVALID, attempts = max
  ACCEPTED --> [*]
  FAILED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryRequestEntity ||--o{ QueryRequestCreated : emits
  QueryRequestEntity ||--o{ AttemptGenerated : emits
  QueryRequestEntity ||--o{ AttemptMutationGuardrailVerdictRecorded : emits
  QueryRequestEntity ||--o{ AttemptValidated : emits
  QueryRequestEntity ||--o{ QueryRequestAccepted : emits
  QueryRequestEntity ||--o{ QueryRequestFailedFinal : emits
  QueryRequestEntity ||--o{ ValidationEvalRecorded : emits
  QueryRequestView }o--|| QueryRequestEntity : projects
  RequestQueue ||--o{ QuestionSubmitted : emits
  SubmissionConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `ValidatorAgent` | `application/ValidatorAgent.java` |
| `SqlTasks` | `application/SqlTasks.java` |
| `SqlWorkflow` | `application/SqlWorkflow.java` |
| `QueryRequestEntity` | `application/QueryRequestEntity.java` (state in `domain/QueryRequest.java`, events in `domain/QueryRequestEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `QueryRequestView` | `application/QueryRequestView.java` |
| `SubmissionConsumer` | `application/SubmissionConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `SqlEndpoint` | `api/SqlEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `validateStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — the workflow degrades to `FAILED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `SqlEndpoint.submit` uses `(question, schemaName, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(requestId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `sql-gen.workflow.max-attempts` (default 4). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism is achieved via `failStep`; it preserves the best query and every validation note on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it scans `query.sql()` for mutating keywords using a case-insensitive whole-word regex and either advances to `validateStep` or returns to `generateStep` with a structured feedback note. The structured feedback never becomes an LLM-generated note; it stays a deterministic `ValidationNotes` payload with a single bullet.
