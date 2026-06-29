# PLAN — code-eval-loop

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
  Ver[VerifierAgent]:::agent

  WF[SolutionWorkflow]:::wf
  Sol[SolutionEntity]:::ese
  Queue[ProblemQueue]:::ese
  View[SolutionsView]:::view
  Consumer[ProblemConsumer]:::cons
  Sim[ProblemSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[CodeEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue problem| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Gen
  WF -->|verify| Ver
  WF -->|emit events| Sol
  Sol -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Sol
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as CodeEndpoint
  participant Q as ProblemQueue
  participant C as ProblemConsumer
  participant W as SolutionWorkflow
  participant G as GeneratorAgent
  participant V as VerifierAgent
  participant E as SolutionEntity
  participant S as SolutionsView

  U->>API: POST /api/solutions {statement, language}
  API->>Q: append ProblemSubmitted
  API-->>U: 202 {solutionId}
  Q->>C: ProblemSubmitted
  C->>W: start({solutionId, statement, language, maxAttempts=5})
  W->>E: emit SolutionCreated (GENERATING)

  W->>G: GENERATE(statement, language)
  G-->>W: CodeAttempt #1 (32 lines)
  W->>E: emit AttemptGenerated (n=1)
  Note over W: sandboxStep (deterministic import/syscall scan)
  W->>E: emit AttemptSandboxVerdictRecorded (passed=true)
  W->>E: status VERIFYING
  W->>V: VERIFY(CodeAttempt #1)
  V-->>W: Verification{FAIL, score=55, 3 diagnostics}
  W->>E: emit AttemptVerified (n=1, FAIL)

  W->>G: REVISE_CODE(statement, prior, notes)
  G-->>W: CodeAttempt #2 (28 lines)
  W->>E: emit AttemptGenerated (n=2)
  W->>E: emit AttemptSandboxVerdictRecorded (passed=true)
  W->>V: VERIFY(CodeAttempt #2)
  V-->>W: Verification{PASS, score=100, rationale}
  W->>E: emit AttemptVerified (n=2, PASS)
  W->>E: emit SolutionPassed (n=2)
  E-->>S: project
  S-->>U: SSE update
```

## State machine — `SolutionEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> VERIFYING: AttemptGenerated + sandbox passed
  GENERATING --> GENERATING: sandbox blocked, re-generate
  VERIFYING --> GENERATING: Verification = FAIL, attempts < max
  VERIFYING --> PASSED: Verification = PASS
  VERIFYING --> EXHAUSTED: Verification = FAIL, attempts = max
  PASSED --> [*]
  EXHAUSTED --> [*]
```

## Entity model

```mermaid
erDiagram
  SolutionEntity ||--o{ SolutionCreated : emits
  SolutionEntity ||--o{ AttemptGenerated : emits
  SolutionEntity ||--o{ AttemptSandboxVerdictRecorded : emits
  SolutionEntity ||--o{ AttemptVerified : emits
  SolutionEntity ||--o{ SolutionPassed : emits
  SolutionEntity ||--o{ SolutionExhausted : emits
  SolutionEntity ||--o{ EvalRecorded : emits
  SolutionsView }o--|| SolutionEntity : projects
  ProblemQueue ||--o{ ProblemSubmitted : emits
  ProblemConsumer }o--|| ProblemQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `VerifierAgent` | `application/VerifierAgent.java` |
| `CodeTasks` | `application/CodeTasks.java` |
| `SolutionWorkflow` | `application/SolutionWorkflow.java` |
| `SolutionEntity` | `application/SolutionEntity.java` (state in `domain/Solution.java`, events in `domain/SolutionEvent.java`) |
| `ProblemQueue` | `application/ProblemQueue.java` |
| `SolutionsView` | `application/SolutionsView.java` |
| `ProblemConsumer` | `application/ProblemConsumer.java` |
| `ProblemSimulator` | `application/ProblemSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `CodeEndpoint` | `api/CodeEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `verifyStep` each carry `stepTimeout(Duration.ofSeconds(90))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep))` — the workflow degrades to `EXHAUSTED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `CodeEndpoint.submit` uses `(statement, submittedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(solutionId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `code-eval-loop.solution.max-attempts` (default 5). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (end-state `EXHAUSTED`) is the only terminal fallback; it preserves the best attempt and every diagnostic on the entity.
- **sandboxStep:** `sandboxStep` is pure-function (no LLM call); it scans the code string for forbidden import patterns (`java.lang.Runtime`, `java.lang.ProcessBuilder`, `sun.*`) and syscall markers, then either advances to `verifyStep` or returns to `generateStep` with a structured VerificationNotes payload. The structured feedback never becomes an LLM-generated diagnostic; it is a deterministic `VerificationNotes` list naming the offending pattern.
- **ci-gate:** after `PASS` is returned by the VerifierAgent, `passStep` runs a final deterministic re-check of the verification score field. Only a score of 100 advances to `SolutionPassed`; partial scores re-enter the loop as `FAIL` to guard against hallucinated pass verdicts.
