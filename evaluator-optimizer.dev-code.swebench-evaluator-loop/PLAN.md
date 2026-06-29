# PLAN — swebench-evaluator-loop

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

  Patch[PatchAgent]:::agent
  Eval[TestEvaluatorAgent]:::agent

  WF[SolvingWorkflow]:::wf
  Issue[IssueEntity]:::ese
  Queue[IssueQueue]:::ese
  View[IssuesView]:::view
  Consumer[IssueIngestConsumer]:::cons
  Sim[IssueSimulator]:::ta
  EvalSamp[EvalSampler]:::ta
  API[IssueEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue issue| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise patch| Patch
  WF -->|evaluate patch| Eval
  WF -->|emit events| Issue
  Issue -.->|projects| View
  API -->|query / SSE| View
  EvalSamp -.->|every 30s| Issue
```

## Interaction sequence — J1 (convergence on iteration 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as IssueEndpoint
  participant Q as IssueQueue
  participant C as IssueIngestConsumer
  participant W as SolvingWorkflow
  participant P as PatchAgent
  participant T as TestEvaluatorAgent
  participant E as IssueEntity
  participant V as IssuesView

  U->>API: POST /api/issues {repoName, issueNumber, description}
  API->>Q: append IssueSubmitted
  API-->>U: 202 {issueId}
  Q->>C: IssueSubmitted
  C->>W: start({issueId, repoName, issueNumber, maxIterations=5})
  W->>E: emit IssueCreated (PATCHING)

  W->>P: GENERATE_PATCH(issueContext)
  P-->>W: PatchAttempt #1 (150 lines)
  W->>E: emit IterationPatched (n=1)
  Note over W: gateStep (deterministic diff validation)
  W->>E: emit IterationGateVerdictRecorded (passed=true)
  W->>E: status EVALUATING
  W->>T: EVALUATE_PATCH(PatchAttempt #1)
  T-->>W: TestResult{FAIL, failCount=2, summary}
  W->>E: emit IterationEvaluated (n=1, FAIL)

  W->>P: REVISE_PATCH(context, prior, failureSummary)
  P-->>W: PatchAttempt #2 (130 lines)
  W->>E: emit IterationPatched (n=2)
  W->>E: emit IterationGateVerdictRecorded (passed=true)
  W->>T: EVALUATE_PATCH(PatchAttempt #2)
  T-->>W: TestResult{PASS, passCount=10, failCount=0}
  W->>E: emit IterationEvaluated (n=2, PASS)
  W->>E: emit IssueSolved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `IssueEntity`

```mermaid
stateDiagram-v2
  [*] --> PATCHING
  PATCHING --> EVALUATING: IterationPatched + gate passed
  PATCHING --> PATCHING: gate blocked, re-patch
  EVALUATING --> PATCHING: TestResult = FAIL, iterations < max
  EVALUATING --> SOLVED: TestResult = PASS
  EVALUATING --> EXHAUSTED: TestResult = FAIL, iterations = max OR token budget hit
  SOLVED --> [*]
  EXHAUSTED --> [*]
```

## Entity model

```mermaid
erDiagram
  IssueEntity ||--o{ IssueCreated : emits
  IssueEntity ||--o{ IterationPatched : emits
  IssueEntity ||--o{ IterationGateVerdictRecorded : emits
  IssueEntity ||--o{ IterationEvaluated : emits
  IssueEntity ||--o{ IssueSolved : emits
  IssueEntity ||--o{ IssueExhausted : emits
  IssueEntity ||--o{ EvalRecorded : emits
  IssuesView }o--|| IssueEntity : projects
  IssueQueue ||--o{ IssueSubmitted : emits
  IssueIngestConsumer }o--|| IssueQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PatchAgent` | `application/PatchAgent.java` |
| `TestEvaluatorAgent` | `application/TestEvaluatorAgent.java` |
| `CodingTasks` | `application/CodingTasks.java` |
| `SolvingWorkflow` | `application/SolvingWorkflow.java` |
| `IssueEntity` | `application/IssueEntity.java` (state in `domain/Issue.java`, events in `domain/IssueEvent.java`) |
| `IssueQueue` | `application/IssueQueue.java` |
| `IssuesView` | `application/IssuesView.java` |
| `IssueIngestConsumer` | `application/IssueIngestConsumer.java` |
| `IssueSimulator` | `application/IssueSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `IssueEndpoint` | `api/IssueEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `patchStep` and `evaluateStep` each carry `stepTimeout(Duration.ofSeconds(120))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4). Patch generation and test evaluation both involve multi-step reasoning that regularly exceeds the default.
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep))` — the workflow degrades to `EXHAUSTED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `IssueEndpoint.submit` deduplicates on `(repoName, issueNumber)` over a 30 s window.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(issueId, iterationNumber)` so a tick that fires twice for the same iteration is a no-op on the entity side.
- **maxIterations ceiling:** read from `swebench.solving.max-iterations` (default 5). The workflow checks the count AND the token budget BEFORE calling `patchStep` for the next iteration; it never recurses past either ceiling.
- **Token budget:** `tokenBudget` (default 50000) tracks cumulative token usage across agent calls in the workflow's local state. The `exhaustStep` is triggered if either ceiling is reached.
- **Gate step:** `gateStep` is pure-function (no LLM call); it validates the unified diff structure and checks `lineCount <= swebench.solving.max-diff-lines`. A malformed patch produces a structured `TestFailureSummary` with a single `errorSnippets` entry; this is fed to `patchStep` as the prior failure summary. Gate-blocked iterations still count toward `maxIterations`.
- **Saga semantics:** there are no external side-effects to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the best patch and every test result on the entity.
