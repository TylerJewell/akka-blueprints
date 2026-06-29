# PLAN — warehouse-optimizer

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

  Opt[OptimizerAgent]:::agent
  Eval[EvaluatorAgent]:::agent

  WF[OptimizationWorkflow]:::wf
  Req[OptimizationRequestEntity]:::ese
  Queue[RequestQueue]:::ese
  View[RequestsView]:::view
  Consumer[RequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  EvalS[EvalSampler]:::ta
  DBA[DBAApprovalGate]:::ta
  API[OptimizerEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue request| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|propose / revise| Opt
  WF -->|evaluate| Eval
  WF -->|emit events| Req
  WF -.->|pause on DDL| DBA
  API -->|DBA decision| WF
  Req -.->|projects| View
  API -->|query / SSE| View
  EvalS -.->|every 30s| Req
  DBA -.->|every 15s reminder| View
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as OptimizerEndpoint
  participant Q as RequestQueue
  participant C as RequestConsumer
  participant W as OptimizationWorkflow
  participant O as OptimizerAgent
  participant EV as EvaluatorAgent
  participant R as OptimizationRequestEntity
  participant V as RequestsView

  U->>API: POST /api/requests {originalSql, objective}
  API->>Q: append RequestSubmitted
  API-->>U: 202 {requestId}
  Q->>C: RequestSubmitted
  C->>W: start({requestId, originalSql, objective, maxAttempts=4})
  W->>R: emit RequestCreated (PROPOSING)

  W->>O: PROPOSE(originalSql, objective)
  O-->>W: Proposal #1 (QUERY_REWRITE)
  W->>R: emit AttemptProposed (n=1)
  Note over W: guardrailStep (deterministic DDL check)
  W->>R: emit AttemptGuardrailVerdictRecorded (passed=true, OK)
  W->>R: status EVALUATING
  W->>EV: EVALUATE(Proposal #1)
  EV-->>W: Evaluation{REVISE, score=3, 3 bullets}
  W->>R: emit AttemptEvaluated (n=1, REVISE)

  W->>O: REVISE_PROPOSAL(originalSql, objective, prior, notes)
  O-->>W: Proposal #2 (QUERY_REWRITE, improved)
  W->>R: emit AttemptProposed (n=2)
  W->>R: emit AttemptGuardrailVerdictRecorded (passed=true, OK)
  W->>EV: EVALUATE(Proposal #2)
  EV-->>W: Evaluation{APPROVE, score=5, rationale}
  W->>R: emit AttemptEvaluated (n=2, APPROVE)
  W->>R: emit RequestApproved (n=2)
  R-->>V: project
  V-->>U: SSE update
```

## State machine — `OptimizationRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> PROPOSING
  PROPOSING --> EVALUATING: guardrail passed (non-DDL)
  PROPOSING --> AWAITING_DBA: guardrail fired (DDL detected)
  PROPOSING --> PROPOSING: guardrail passed, evaluator REVISE, attempts < max
  AWAITING_DBA --> EVALUATING: DBA approved
  AWAITING_DBA --> REJECTED_FINAL: DBA rejected
  EVALUATING --> PROPOSING: Evaluation = REVISE, attempts < max
  EVALUATING --> APPROVED: Evaluation = APPROVE
  EVALUATING --> REJECTED_FINAL: Evaluation = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  OptimizationRequestEntity ||--o{ RequestCreated : emits
  OptimizationRequestEntity ||--o{ AttemptProposed : emits
  OptimizationRequestEntity ||--o{ AttemptGuardrailVerdictRecorded : emits
  OptimizationRequestEntity ||--o{ DBADecisionRecorded : emits
  OptimizationRequestEntity ||--o{ AttemptEvaluated : emits
  OptimizationRequestEntity ||--o{ RequestApproved : emits
  OptimizationRequestEntity ||--o{ RequestRejectedFinal : emits
  OptimizationRequestEntity ||--o{ EvalRecorded : emits
  RequestsView }o--|| OptimizationRequestEntity : projects
  RequestQueue ||--o{ RequestSubmitted : emits
  RequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `OptimizerAgent` | `application/OptimizerAgent.java` |
| `EvaluatorAgent` | `application/EvaluatorAgent.java` |
| `WarehouseTasks` | `application/WarehouseTasks.java` |
| `OptimizationWorkflow` | `application/OptimizationWorkflow.java` |
| `OptimizationRequestEntity` | `application/OptimizationRequestEntity.java` (state in `domain/OptimizationRequest.java`, events in `domain/RequestEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `RequestsView` | `application/RequestsView.java` |
| `RequestConsumer` | `application/RequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `DBAApprovalGate` | `application/DBAApprovalGate.java` |
| `OptimizerEndpoint` | `api/OptimizerEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `proposeStep` and `evaluateStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **DBA gate pause:** `dbaGateStep` has no LLM call and no step timeout — it awaits an external signal indefinitely. The `DBAApprovalGate` TimedAction logs a reminder every 15 s for requests that have been in `AWAITING_DBA` for more than 5 minutes; it does not auto-approve or auto-reject.
- **Idempotency:** `OptimizerEndpoint.submit` uses `(originalSql, submittedBy)` over a 10 s window as the dedup key. `EvalSampler` deduplicates on `(requestId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `warehouse-optimizer.optimization.max-attempts` (default 4). The workflow checks the count BEFORE calling `proposeStep` for the next iteration; it never recurses past the ceiling.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it inspects the proposal text for DDL keywords using a case-insensitive regex and either advances to `evaluateStep` (non-DDL) or transitions to `dbaGateStep` (DDL detected).
- **Saga semantics:** there is no external warehouse execution; proposals are recommendations only. The halt mechanism (`HT1`) and DBA-rejection path are the only "compensations"; both preserve every proposal and evaluation on the entity.
