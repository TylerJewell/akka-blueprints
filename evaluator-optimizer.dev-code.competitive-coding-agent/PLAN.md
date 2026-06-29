# PLAN — competitive-coding-agent

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
  classDef ext fill:#0f1a10,stroke:#F5C518,color:#F5C518,stroke-dasharray:4 2;

  Solver[SolverAgent]:::agent
  Judge[JudgeAgent]:::agent

  WF[SolvingWorkflow]:::wf
  Sub[SubmissionEntity]:::ese
  Queue[ProblemQueue]:::ese
  View[SubmissionsView]:::view
  Consumer[ProblemConsumer]:::cons
  Sim[ProblemSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[SolverEndpoint]:::ep
  App[AppEndpoint]:::ep
  E2B[E2B Sandbox]:::ext

  API -->|enqueue problem| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Solver
  WF -->|execute in sandbox| E2B
  WF -->|evaluate report| Judge
  WF -->|emit events| Sub
  Sub -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 45s| Sub
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SolverEndpoint
  participant Q as ProblemQueue
  participant C as ProblemConsumer
  participant W as SolvingWorkflow
  participant S as SolverAgent
  participant SB as E2B Sandbox
  participant J as JudgeAgent
  participant E as SubmissionEntity
  participant V as SubmissionsView

  U->>API: POST /api/submissions {title, statementText, sampleCases}
  API->>Q: append ProblemSubmitted
  API-->>U: 202 {submissionId}
  Q->>C: ProblemSubmitted
  C->>W: start({submissionId, title, statementText, maxAttempts=5})
  W->>E: emit SubmissionCreated (GENERATING)

  W->>S: GENERATE(problem)
  S-->>W: GeneratedSolution #1
  W->>E: emit AttemptGenerated (n=1)
  Note over W: sandboxStep (resource-bounded execution)
  W->>SB: compile + run against test cases
  SB-->>W: SandboxReport{resourcesOk=true, caseResults}
  W->>E: emit AttemptSandboxReportRecorded (passed=true)
  W->>E: status JUDGING
  W->>J: EVALUATE_SANDBOX(SandboxReport)
  J-->>W: JudgeVerdict{FAIL, 2/3 cases, 3 bullets}
  W->>E: emit AttemptJudged (n=1, FAIL)

  W->>S: REVISE_SOLUTION(problem, priorSolution, notes)
  S-->>W: GeneratedSolution #2
  W->>E: emit AttemptGenerated (n=2)
  W->>SB: compile + run against test cases
  SB-->>W: SandboxReport{resourcesOk=true, all cases pass}
  W->>E: emit AttemptSandboxReportRecorded (passed=true)
  W->>J: EVALUATE_SANDBOX(SandboxReport)
  J-->>W: JudgeVerdict{PASS, 3/3 cases, rationale}
  W->>E: emit AttemptJudged (n=2, PASS)
  W->>E: emit SubmissionAccepted (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `SubmissionEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> JUDGING: AttemptGenerated + sandbox resourcesOk
  GENERATING --> GENERATING: sandbox blocked (TLE/MLE), re-generate
  JUDGING --> GENERATING: JudgeVerdict = FAIL, attempts < max
  JUDGING --> ACCEPTED: JudgeVerdict = PASS
  JUDGING --> REJECTED_FINAL: JudgeVerdict = FAIL, attempts = max
  ACCEPTED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  SubmissionEntity ||--o{ SubmissionCreated : emits
  SubmissionEntity ||--o{ AttemptGenerated : emits
  SubmissionEntity ||--o{ AttemptSandboxReportRecorded : emits
  SubmissionEntity ||--o{ AttemptJudged : emits
  SubmissionEntity ||--o{ SubmissionAccepted : emits
  SubmissionEntity ||--o{ SubmissionRejectedFinal : emits
  SubmissionEntity ||--o{ EvalRecorded : emits
  SubmissionsView }o--|| SubmissionEntity : projects
  ProblemQueue ||--o{ ProblemSubmitted : emits
  ProblemConsumer }o--|| ProblemQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SolverAgent` | `application/SolverAgent.java` |
| `JudgeAgent` | `application/JudgeAgent.java` |
| `SolverTasks` | `application/SolverTasks.java` |
| `SolvingWorkflow` | `application/SolvingWorkflow.java` |
| `SubmissionEntity` | `application/SubmissionEntity.java` (state in `domain/Submission.java`, events in `domain/SubmissionEvent.java`) |
| `ProblemQueue` | `application/ProblemQueue.java` |
| `SubmissionsView` | `application/SubmissionsView.java` |
| `ProblemConsumer` | `application/ProblemConsumer.java` |
| `ProblemSimulator` | `application/ProblemSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `SolverEndpoint` | `api/SolverEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `sandboxStep` each carry `stepTimeout(Duration.ofSeconds(120))`; `judgeStep` carries `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling or sandbox-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent or sandbox failure rather than hanging.
- **Idempotency:** `SolverEndpoint.submit` uses `(title, submittedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(submissionId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `competitive-coding-agent.solving.max-attempts` (default 5). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Sandbox step:** `sandboxStep` is a pure-infrastructure step (no LLM call). It POSTs the generated source to the E2B HTTP API, awaits the execution result, and deserialises it into a `SandboxReport`. The E2B API key is read from `${?E2B_API_KEY}` at runtime; the workflow fails fast with a clear message if the key is absent.
- **Saga semantics:** the sandbox execution has no durable side-effects to compensate. If the sandbox call fails transiently, `maxRetries(2)` retries it in-process before failing over to `rejectStep`.
- **Best-of selection on rejection:** `rejectStep` selects the attempt whose `passedCases / totalCases` ratio is highest. Ties are broken by `attemptNumber` (later attempt preferred). The source code of that attempt is stored as `acceptedSourceCode` with `rejectionReason = "max attempts reached (N)"`.
