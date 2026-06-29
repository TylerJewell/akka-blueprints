# PLAN — llm-judge-loop

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
  Judge[JudgeAgent]:::agent

  WF[EvaluationWorkflow]:::wf
  Eval[EvaluationEntity]:::ese
  Queue[QuestionQueue]:::ese
  View[EvaluationsView]:::view
  Consumer[QuestionConsumer]:::cons
  Sim[QuestionSimulator]:::ta
  Sampler[JudgeSampler]:::ta
  API[EvaluationEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit question| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Gen
  WF -->|judge| Judge
  WF -->|emit events| Eval
  Eval -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 30s| Eval
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as EvaluationEndpoint
  participant Q as QuestionQueue
  participant C as QuestionConsumer
  participant W as EvaluationWorkflow
  participant G as GeneratorAgent
  participant J as JudgeAgent
  participant E as EvaluationEntity
  participant V as EvaluationsView

  U->>API: POST /api/evaluations {questionText, scoreThreshold}
  API->>Q: append QuestionSubmitted
  API-->>U: 202 {evaluationId}
  Q->>C: QuestionSubmitted
  C->>W: start({evaluationId, questionText, scoreThreshold, maxAttempts=4})
  W->>E: emit EvaluationCreated (GENERATING)

  W->>G: GENERATE(questionText, scoreThreshold)
  G-->>W: GeneratedAnswer #1 (120 tokens)
  W->>E: emit AttemptGenerated (n=1)
  Note over W: guardrailStep (deterministic structural check)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>E: status JUDGING
  W->>J: JUDGE(GeneratedAnswer #1)
  J-->>W: Judgment{REVISE, score=3, 3 bullets}
  W->>E: emit AttemptJudged (n=1, REVISE)

  W->>G: REVISE_ANSWER(questionText, prior, feedback)
  G-->>W: GeneratedAnswer #2 (180 tokens)
  W->>E: emit AttemptGenerated (n=2)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>J: JUDGE(GeneratedAnswer #2)
  J-->>W: Judgment{ACCEPT, score=5, rationale}
  W->>E: emit AttemptJudged (n=2, ACCEPT)
  W->>E: emit EvaluationAccepted (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `EvaluationEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> JUDGING: AttemptGenerated + guardrail passed
  GENERATING --> GENERATING: guardrail blocked, re-generate
  JUDGING --> GENERATING: Judgment = REVISE, attempts < max
  JUDGING --> ACCEPTED: Judgment = ACCEPT
  JUDGING --> REJECTED_FINAL: Judgment = REVISE, attempts = max
  ACCEPTED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  EvaluationEntity ||--o{ EvaluationCreated : emits
  EvaluationEntity ||--o{ AttemptGenerated : emits
  EvaluationEntity ||--o{ AttemptGuardrailVerdictRecorded : emits
  EvaluationEntity ||--o{ AttemptJudged : emits
  EvaluationEntity ||--o{ EvaluationAccepted : emits
  EvaluationEntity ||--o{ EvaluationRejectedFinal : emits
  EvaluationEntity ||--o{ JudgmentRecorded : emits
  EvaluationsView }o--|| EvaluationEntity : projects
  QuestionQueue ||--o{ QuestionSubmitted : emits
  QuestionConsumer }o--|| QuestionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `JudgeAgent` | `application/JudgeAgent.java` |
| `JudgeTasks` | `application/JudgeTasks.java` |
| `EvaluationWorkflow` | `application/EvaluationWorkflow.java` |
| `EvaluationEntity` | `application/EvaluationEntity.java` (state in `domain/Evaluation.java`, events in `domain/EvaluationEvent.java`) |
| `QuestionQueue` | `application/QuestionQueue.java` |
| `EvaluationsView` | `application/EvaluationsView.java` |
| `QuestionConsumer` | `application/QuestionConsumer.java` |
| `QuestionSimulator` | `application/QuestionSimulator.java` |
| `JudgeSampler` | `application/JudgeSampler.java` |
| `EvaluationEndpoint` | `api/EvaluationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `judgeStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `EvaluationEndpoint.submit` uses `(questionText, submittedBy)` over a 10 s window as the dedup key.
- **JudgeSampler idempotency:** the sampler keys its `recordJudgmentEvent` calls on `(evaluationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `llm-judge-loop.evaluation.max-attempts` (default 4). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt on ceiling exhaustion ends in `REJECTED_FINAL`, preserving the best answer and every judgment on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it checks `answer.tokenCount() >= minTokens` and `!answer.text().contains(REFUSED_SENTINEL)`, and either advances to `judgeStep` or returns to `generateStep` with a structured feedback note.
