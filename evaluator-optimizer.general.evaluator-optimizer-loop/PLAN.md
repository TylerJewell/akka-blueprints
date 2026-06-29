# PLAN — evaluator-optimizer-loop

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
  Eval[EvaluatorAgent]:::agent

  WF[OptimizationWorkflow]:::wf
  Job[JobEntity]:::ese
  Queue[SubmissionQueue]:::ese
  View[JobsView]:::view
  Consumer[SubmissionConsumer]:::cons
  Sim[ProblemSimulator]:::ta
  Sampler[EvalSampler]:::ta
  API[OptimizationEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit problem| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Gen
  WF -->|evaluate| Eval
  WF -->|emit events| Job
  Job -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 30s| Job
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as OptimizationEndpoint
  participant Q as SubmissionQueue
  participant C as SubmissionConsumer
  participant W as OptimizationWorkflow
  participant G as GeneratorAgent
  participant EV as EvaluatorAgent
  participant J as JobEntity
  participant V as JobsView

  U->>API: POST /api/jobs {description, tokenCeiling}
  API->>Q: append ProblemSubmitted
  API-->>U: 202 {jobId}
  Q->>C: ProblemSubmitted
  C->>W: start({jobId, description, tokenCeiling, maxAttempts=4})
  W->>J: emit JobCreated (GENERATING)

  W->>G: GENERATE(description, tokenCeiling)
  G-->>W: Candidate #1 (480 tokens)
  W->>J: emit AttemptGenerated (n=1)
  Note over W: guardrailStep (deterministic token check)
  W->>J: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>J: status EVALUATING
  W->>EV: EVALUATE(Candidate #1)
  EV-->>W: Evaluation{REVISE, score=3, 3 bullets}
  W->>J: emit AttemptEvaluated (n=1, REVISE)

  W->>G: REVISE_CANDIDATE(description, prior, notes)
  G-->>W: Candidate #2 (440 tokens)
  W->>J: emit AttemptGenerated (n=2)
  W->>J: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>EV: EVALUATE(Candidate #2)
  EV-->>W: Evaluation{ACCEPT, score=5, rationale}
  W->>J: emit AttemptEvaluated (n=2, ACCEPT)
  W->>J: emit JobAccepted (n=2)
  J-->>V: project
  V-->>U: SSE update
```

## State machine — `JobEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> EVALUATING: AttemptGenerated + guardrail passed
  GENERATING --> GENERATING: guardrail blocked, re-generate
  EVALUATING --> GENERATING: Evaluation = REVISE, attempts < max
  EVALUATING --> ACCEPTED: Evaluation = ACCEPT
  EVALUATING --> REJECTED_FINAL: Evaluation = REVISE, attempts = max
  ACCEPTED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  JobEntity ||--o{ JobCreated : emits
  JobEntity ||--o{ AttemptGenerated : emits
  JobEntity ||--o{ AttemptGuardrailVerdictRecorded : emits
  JobEntity ||--o{ AttemptEvaluated : emits
  JobEntity ||--o{ JobAccepted : emits
  JobEntity ||--o{ JobRejectedFinal : emits
  JobEntity ||--o{ EvalRecorded : emits
  JobsView }o--|| JobEntity : projects
  SubmissionQueue ||--o{ ProblemSubmitted : emits
  SubmissionConsumer }o--|| SubmissionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `EvaluatorAgent` | `application/EvaluatorAgent.java` |
| `OptimizationTasks` | `application/OptimizationTasks.java` |
| `OptimizationWorkflow` | `application/OptimizationWorkflow.java` |
| `JobEntity` | `application/JobEntity.java` (state in `domain/Job.java`, events in `domain/JobEvent.java`) |
| `SubmissionQueue` | `application/SubmissionQueue.java` |
| `JobsView` | `application/JobsView.java` |
| `SubmissionConsumer` | `application/SubmissionConsumer.java` |
| `ProblemSimulator` | `application/ProblemSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `OptimizationEndpoint` | `api/OptimizationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `evaluateStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `OptimizationEndpoint.submit` uses `(description, submittedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(jobId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `evaluator-optimizer.loop.max-attempts` (default 4). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. On irrecoverable failure the `defaultStepRecovery` failover lands in `REJECTED_FINAL`, preserving every candidate and every evaluation on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it computes the token count from the candidate and either advances to `evaluateStep` or returns to `generateStep` with a structured feedback note. The structured feedback is a deterministic `EvaluationNotes` payload with a single bullet.
