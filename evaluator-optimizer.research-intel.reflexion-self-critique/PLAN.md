# PLAN — reflexion-self-critique

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

  Actor[ActorAgent]:::agent
  Reflex[ReflexionAgent]:::agent

  WF[ReflexionWorkflow]:::wf
  QE[QueryEntity]:::ese
  QQ[QueryQueue]:::ese
  View[QueriesView]:::view
  Consumer[QueryConsumer]:::cons
  Sim[QuerySimulator]:::ta
  Sampler[ReflexionSampler]:::ta
  API[ResearchEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue question| QQ
  Sim -.->|every 60s| QQ
  QQ -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|answer / retry-answer| Actor
  WF -->|reflect| Reflex
  WF -->|emit events| QE
  QE -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 30s| QE
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ResearchEndpoint
  participant QQ as QueryQueue
  participant C as QueryConsumer
  participant W as ReflexionWorkflow
  participant A as ActorAgent
  participant R as ReflexionAgent
  participant QE as QueryEntity
  participant V as QueriesView

  U->>API: POST /api/queries {questionText, citationFloor}
  API->>QQ: append QuestionSubmitted
  API-->>U: 202 {queryId}
  QQ->>C: QuestionSubmitted
  C->>W: start({queryId, questionText, citationFloor, maxAttempts=4})
  W->>QE: emit QueryCreated (RESEARCHING)

  W->>A: ANSWER(questionText, citationFloor)
  Note over A: searchDocuments tool call → retrieve SourceDocuments
  A-->>W: CandidateAnswer #1 (2 citations)
  W->>QE: emit AttemptAnswered (n=1)
  Note over W: guardrailStep (deterministic citation check)
  W->>QE: emit AttemptCitationGuardrailRecorded (passed=true)
  W->>QE: status REFLECTING
  W->>R: REFLECT(CandidateAnswer #1)
  R-->>W: Reflexion{RETRY, score=2, reinforcementParagraph, 3 bullets}
  W->>QE: emit AttemptReflected (n=1, RETRY)

  W->>A: RETRY_ANSWER(questionText, priorAnswer, ReflexionNote)
  Note over A: searchDocuments again, guided by verbal memory
  A-->>W: CandidateAnswer #2 (4 citations)
  W->>QE: emit AttemptAnswered (n=2)
  W->>QE: emit AttemptCitationGuardrailRecorded (passed=true)
  W->>R: REFLECT(CandidateAnswer #2)
  R-->>W: Reflexion{PASS, score=5, rationale}
  W->>QE: emit AttemptReflected (n=2, PASS)
  W->>QE: emit QueryResolved (n=2)
  QE-->>V: project
  V-->>U: SSE update
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> RESEARCHING
  RESEARCHING --> REFLECTING: AttemptAnswered + citation guardrail passed
  RESEARCHING --> RESEARCHING: citation guardrail blocked, re-answer
  REFLECTING --> RESEARCHING: Reflexion = RETRY, attempts < max
  REFLECTING --> RESOLVED: Reflexion = PASS
  REFLECTING --> EXHAUSTED: Reflexion = RETRY, attempts = max
  RESOLVED --> [*]
  EXHAUSTED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QueryCreated : emits
  QueryEntity ||--o{ AttemptAnswered : emits
  QueryEntity ||--o{ AttemptCitationGuardrailRecorded : emits
  QueryEntity ||--o{ AttemptReflected : emits
  QueryEntity ||--o{ QueryResolved : emits
  QueryEntity ||--o{ QueryExhausted : emits
  QueryEntity ||--o{ ReflexionRecorded : emits
  QueriesView }o--|| QueryEntity : projects
  QueryQueue ||--o{ QuestionSubmitted : emits
  QueryConsumer }o--|| QueryQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ActorAgent` | `application/ActorAgent.java` |
| `ReflexionAgent` | `application/ReflexionAgent.java` |
| `ResearchTasks` | `application/ResearchTasks.java` |
| `ReflexionWorkflow` | `application/ReflexionWorkflow.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `QueryQueue` | `application/QueryQueue.java` |
| `QueriesView` | `application/QueriesView.java` |
| `QueryConsumer` | `application/QueryConsumer.java` |
| `QuerySimulator` | `application/QuerySimulator.java` |
| `ReflexionSampler` | `application/ReflexionSampler.java` |
| `ResearchEndpoint` | `api/ResearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `answerStep` and `reflectStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep))` — the workflow degrades to `EXHAUSTED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `ResearchEndpoint.submit` uses `(questionText, submittedBy)` over a 10 s window as the dedup key.
- **ReflexionSampler idempotency:** the sampler keys its `recordReflexionEval` calls on `(queryId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `reflexion.max-attempts` (default 4). The workflow checks the count BEFORE calling `answerStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the best answer and every reflexion note on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it checks `answer.citationCount() >= citationFloor` and either advances to `reflectStep` or returns to `answerStep` with a structured feedback `ReflexionNote`. The feedback is never an LLM-generated critique; it is a deterministic message with a single focusBullet naming the shortfall count.
- **Verbal memory propagation:** on `RETRY_ANSWER`, the full `ReflexionNote` — `reinforcementParagraph` plus `focusBullets` — is passed as a structured input to `ActorAgent`. The agent serialises this into its prompt context as prior reasoning, not as a system instruction, so subsequent model calls see it as part of the conversation history.
