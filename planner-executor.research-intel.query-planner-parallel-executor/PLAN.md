# PLAN — query-planner-parallel-executor

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

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

  Planner[PlannerAgent]:::agent
  Corpus[CorpusSearchExecutor]:::agent
  Web[WebLookupExecutor]:::agent
  KB[KnowledgeBaseExecutor]:::agent
  Synth[SynthesisAgent]:::agent

  WF[QueryWorkflow]:::wf
  Session[QuerySessionEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[QueryQueue]:::ese
  View[SessionView]:::view
  Consumer[QueryRequestConsumer]:::cons
  Sim[QuerySimulator]:::ta
  Stale[StaleSessionMonitor]:::ta
  API[QueryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|DECOMPOSE / ASSESS_COVERAGE| Planner
  WF -->|CORPUS_SEARCH| Corpus
  WF -->|WEB_LOOKUP| Web
  WF -->|KB_LOOKUP| KB
  WF -->|SYNTHESIZE_RESULTS| Synth
  WF -->|emit events| Session
  WF -->|poll| Ctrl
  Session -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| Session
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as QueryEndpoint
  participant Q as QueryQueue
  participant C as QueryRequestConsumer
  participant W as QueryWorkflow
  participant P as PlannerAgent
  participant EV as PlanQualityEvaluator
  participant EX as Executors (Corpus/Web/KB)
  participant S as SynthesisAgent
  participant E as QuerySessionEntity
  participant CTL as SystemControlEntity
  participant V as SessionView

  U->>API: POST /api/sessions {question}
  API->>Q: append QuerySubmitted
  API-->>U: 202 {sessionId}
  Q->>C: QuerySubmitted
  C->>W: start({sessionId, question})
  W->>E: emit SessionCreated (PLANNING)
  W->>P: DECOMPOSE(question)
  P-->>W: QueryPlan
  W->>E: emit SessionPlanned (EVALUATING)
  W->>EV: score(plan)
  EV-->>W: PlanEvaluation {passing=true}
  W->>E: emit PlanEvaluated
  loop per round until Sufficient | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>W: guardrail.vet(subQuery) per sub-query
    par fanOut
      W->>EX: CORPUS_SEARCH(subQuery1)
      W->>EX: WEB_LOOKUP(subQuery2)
      W->>EX: KB_LOOKUP(subQuery3)
    end
    EX-->>W: SubQueryResult × N
    W->>W: SecretScrubber.scrub(content) per result
    W->>E: emit SubQueryRecorded × N (EXECUTING)
    W->>P: ASSESS_COVERAGE(plan, results)
    P-->>W: Sufficient
  end
  W->>E: emit CoverageAssessed (SYNTHESIZING)
  W->>S: SYNTHESIZE_RESULTS(question, results)
  S-->>W: ResearchAnswer
  W->>E: emit SessionCompleted (COMPLETED)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `QuerySessionEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EVALUATING: SessionPlanned
  EVALUATING --> EVALUATING: PlanRevisionRequested
  EVALUATING --> EXECUTING: PlanEvaluated(passing)
  EVALUATING --> FAILED: PlanEvaluated(failing×2)
  EXECUTING --> EXECUTING: SubQueryDispatched / SubQueryBlocked / SubQueryRecorded
  EXECUTING --> EVALUATING: CoverageAssessed(NeedsFollowup)
  EXECUTING --> SYNTHESIZING: CoverageAssessed(Sufficient)
  EXECUTING --> FAILED: CoverageAssessed(Fail)
  EXECUTING --> HALTED: SessionHaltedOperator
  EXECUTING --> STALE: SessionTimedOut
  SYNTHESIZING --> COMPLETED: SessionCompleted
  SYNTHESIZING --> FAILED: SessionFailed
  STALE --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  QuerySessionEntity ||--o{ SessionPlanned : emits
  QuerySessionEntity ||--o{ PlanEvaluated : emits
  QuerySessionEntity ||--o{ PlanRevisionRequested : emits
  QuerySessionEntity ||--o{ SubQueryDispatched : emits
  QuerySessionEntity ||--o{ SubQueryBlocked : emits
  QuerySessionEntity ||--o{ SubQueryRecorded : emits
  QuerySessionEntity ||--o{ CoverageAssessed : emits
  QuerySessionEntity ||--o{ SynthesisStarted : emits
  QuerySessionEntity ||--o{ SessionCompleted : emits
  QuerySessionEntity ||--o{ SessionFailed : emits
  QuerySessionEntity ||--o{ SessionHaltedOperator : emits
  QuerySessionEntity ||--o{ SessionTimedOut : emits
  SessionView }o--|| QuerySessionEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  QueryQueue ||--o{ QuerySubmitted : emits
  QueryRequestConsumer }o--|| QueryQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `CorpusSearchExecutor` | `application/CorpusSearchExecutor.java` |
| `WebLookupExecutor` | `application/WebLookupExecutor.java` |
| `KnowledgeBaseExecutor` | `application/KnowledgeBaseExecutor.java` |
| `SynthesisAgent` | `application/SynthesisAgent.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `QuerySessionEntity` | `application/QuerySessionEntity.java` (state in `domain/QuerySession.java`, events in `domain/SessionEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `QueryQueue` | `application/QueryQueue.java` |
| `SessionView` | `application/SessionView.java` |
| `QueryRequestConsumer` | `application/QueryRequestConsumer.java` |
| `QuerySimulator` | `application/QuerySimulator.java` |
| `StaleSessionMonitor` | `application/StaleSessionMonitor.java` |
| `SubQueryGuardrail` | `application/SubQueryGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `PlanQualityEvaluator` | `application/PlanQualityEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Fan-out:** all sub-queries in a round are dispatched in parallel via `CompletableFuture.allOf`. The `fanOutStep` timeout (180 s) covers the full round, not a single executor call.
- **Workflow step timeouts:** `planStep` 60 s, `evalStep` 30 s, `fanOutStep` 180 s, `coverageStep` 45 s, `synthesisStep` 90 s. Default recovery: `maxRetries(2).failoverTo(QueryWorkflow::error)`.
- **Round budget:** at most three rounds. A third coverage evaluation returning `NeedsFollowup` is treated as `Fail`.
- **Plan revision budget:** at most two consecutive low-quality plan evaluations. A third triggers `SessionFailed`.
- **Halt poll:** `checkHaltStep` reads `SystemControlEntity.get` synchronously before each round's fan-out. An operator halt arriving during `fanOutStep` lets the in-flight round finish; the loop exits at the next `checkHaltStep`.
- **Stale detection:** `StaleSessionMonitor` ticks every 30 s; `SessionTimedOut` is non-fatal to other sessions. The workflow's `coverageStep` checks entity status and exits when it reads `STALE`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure; no external state. Same input always yields same scrubbed output, keeping `SubQueryRecorded` events deterministic and replayable.
- **Guardrail determinism:** `SubQueryGuardrail.vet` is pure. Rejection is logged as a `SubQueryBlocked` event; the planner sees the blocked sub-queries in its next `ASSESS_COVERAGE` call.
