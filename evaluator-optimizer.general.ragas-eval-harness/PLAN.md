# PLAN — ragas-eval-harness

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

  Retrieval[RetrievalAgent]:::agent
  Ragas[RagasEvaluatorAgent]:::agent

  WF[EvalWorkflow]:::wf
  Run[EvalRunEntity]:::ese
  Queue[QuestionQueue]:::ese
  View[EvalRunsView]:::view
  Consumer[QuestionConsumer]:::cons
  Sim[QuestionSimulator]:::ta
  Sampler[ScoreSampler]:::ta
  API[EvalEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit question| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|answer / revise| Retrieval
  WF -->|evaluate| Ragas
  WF -->|emit events| Run
  Run -.->|projects| View
  API -->|query / SSE / CI gate| View
  Sampler -.->|every 30s| Run
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as EvalEndpoint
  participant Q as QuestionQueue
  participant C as QuestionConsumer
  participant W as EvalWorkflow
  participant R as RetrievalAgent
  participant E as RagasEvaluatorAgent
  participant EN as EvalRunEntity
  participant V as EvalRunsView

  U->>API: POST /api/runs {question, corpusTag}
  API->>Q: append QuestionSubmitted
  API-->>U: 202 {runId}
  Q->>C: QuestionSubmitted
  C->>W: start({runId, question, maxAttempts=3})
  W->>EN: emit RunCreated (ANSWERING)

  W->>R: ANSWER(question, corpusTag)
  R-->>W: AnswerRecord #1 (confidence=0.75, 3 chunks)
  W->>EN: emit AttemptAnswered (n=1)
  Note over W: groundingStep (deterministic confidence check)
  W->>EN: emit AttemptGroundingVerdictRecorded (passed=true)
  W->>EN: status EVALUATING
  W->>E: EVALUATE(question, AnswerRecord #1)
  E-->>W: RagasScore{RETRY, faithfulness=0.60, answerRelevance=0.72, contextPrecision=0.68}
  W->>EN: emit AttemptScored (n=1, RETRY)

  W->>R: REVISE_ANSWER(question, prior, feedback)
  R-->>W: AnswerRecord #2 (confidence=0.88, 2 chunks)
  W->>EN: emit AttemptAnswered (n=2)
  W->>EN: emit AttemptGroundingVerdictRecorded (passed=true)
  W->>E: EVALUATE(question, AnswerRecord #2)
  E-->>W: RagasScore{PASS, faithfulness=0.91, answerRelevance=0.87, contextPrecision=0.82}
  W->>EN: emit AttemptScored (n=2, PASS)
  W->>EN: emit RunPassed (n=2)
  EN-->>V: project
  V-->>U: SSE update
```

## State machine — `EvalRunEntity`

```mermaid
stateDiagram-v2
  [*] --> ANSWERING
  ANSWERING --> EVALUATING: AttemptAnswered + grounding passed
  ANSWERING --> ANSWERING: grounding blocked, re-answer
  EVALUATING --> ANSWERING: Score = RETRY, attempts < max
  EVALUATING --> PASSED: Score = PASS
  EVALUATING --> FAILED_FINAL: Score = RETRY, attempts = max
  PASSED --> [*]
  FAILED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  EvalRunEntity ||--o{ RunCreated : emits
  EvalRunEntity ||--o{ AttemptAnswered : emits
  EvalRunEntity ||--o{ AttemptGroundingVerdictRecorded : emits
  EvalRunEntity ||--o{ AttemptScored : emits
  EvalRunEntity ||--o{ RunPassed : emits
  EvalRunEntity ||--o{ RunFailedFinal : emits
  EvalRunEntity ||--o{ ScoreRecorded : emits
  EvalRunsView }o--|| EvalRunEntity : projects
  QuestionQueue ||--o{ QuestionSubmitted : emits
  QuestionConsumer }o--|| QuestionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RetrievalAgent` | `application/RetrievalAgent.java` |
| `RagasEvaluatorAgent` | `application/RagasEvaluatorAgent.java` |
| `EvalTasks` | `application/EvalTasks.java` |
| `EvalWorkflow` | `application/EvalWorkflow.java` |
| `EvalRunEntity` | `application/EvalRunEntity.java` (state in `domain/EvalRun.java`, events in `domain/EvalRunEvent.java`) |
| `QuestionQueue` | `application/QuestionQueue.java` |
| `EvalRunsView` | `application/EvalRunsView.java` |
| `QuestionConsumer` | `application/QuestionConsumer.java` |
| `QuestionSimulator` | `application/QuestionSimulator.java` |
| `ScoreSampler` | `application/ScoreSampler.java` |
| `EvalEndpoint` | `api/EvalEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `answerStep` and `scoreStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — the workflow degrades to `FAILED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `EvalEndpoint.submit` uses `(questionText, submittedBy)` over a 10 s window as the dedup key.
- **ScoreSampler idempotency:** the sampler keys its `recordEval` calls on `(runId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `ragas-eval.workflow.max-attempts` (default 3). The workflow checks the count BEFORE calling `answerStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism is the only "compensation"; it preserves the best-scoring answer and every score set on the entity.
- **Grounding step:** `groundingStep` is pure-function (no LLM call); it compares `answer.groundingConfidence()` against the configured floor and either advances to `scoreStep` or returns to `answerStep` with a structured `RagasFeedback` payload. The feedback never becomes an LLM-generated score; it stays a deterministic payload with `failedMetrics=["grounding"]`.
- **CI gate:** `GET /api/ci/gate-status` queries `EvalRunsView.getAllRuns`, counts the last 100 completed runs, and returns `{ gatePassed, passRate, windowSize }`. The endpoint is synchronous and read-only; it does not mutate entity state.
