# PLAN — self-improving-deep-researcher

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

  Researcher[ResearchAgent]:::agent
  Evaluator[EvaluatorAgent]:::agent
  Rewriter[PromptRewriterAgent]:::agent

  WF[ResearchSessionWorkflow]:::wf
  SE[SessionEntity]:::ese
  QQ[QueryQueue]:::ese
  SV[SessionsView]:::view
  QC[QueryConsumer]:::cons
  QSim[QuerySimulator]:::ta
  DS[DriftSampler]:::ta
  API[ResearchEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue query| QQ
  QSim -.->|every 90s| QQ
  QQ -.->|subscribes| QC
  QC -->|start workflow| WF
  WF -->|research / refine| Researcher
  WF -->|evaluate report| Evaluator
  WF -->|rewrite memory| Rewriter
  WF -->|emit events| SE
  SE -.->|projects| SV
  API -->|query / SSE| SV
  DS -.->|every 60s| SE
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ResearchEndpoint
  participant QQ as QueryQueue
  participant QC as QueryConsumer
  participant WF as ResearchSessionWorkflow
  participant RA as ResearchAgent
  participant EA as EvaluatorAgent
  participant PRA as PromptRewriterAgent
  participant SE as SessionEntity
  participant SV as SessionsView

  U->>API: POST /api/sessions {topic, depthHint}
  API->>QQ: append QuerySubmitted
  API-->>U: 202 {sessionId}
  QQ->>QC: QuerySubmitted
  QC->>WF: start({sessionId, topic, depthHint, maxAttempts=3})
  WF->>SE: emit SessionCreated (RESEARCHING)

  WF->>RA: RESEARCH(topic, depthHint, promptMemory)
  RA-->>WF: ResearchReport #1 (4 evidence segments)
  WF->>SE: emit AttemptResearched (n=1)
  WF->>SE: status EVALUATING
  WF->>EA: EVALUATE_REPORT(ResearchReport #1)
  EA-->>WF: QualityEvaluation{REFINE, score=3, 3 bullets}
  WF->>SE: emit AttemptEvaluated (n=1, REFINE)

  WF->>RA: REFINE_RESEARCH(topic, report#1, QualityNotes)
  RA-->>WF: ResearchReport #2 (6 evidence segments)
  WF->>SE: emit AttemptResearched (n=2)
  WF->>SE: status EVALUATING
  WF->>EA: EVALUATE_REPORT(ResearchReport #2)
  EA-->>WF: QualityEvaluation{SUFFICIENT, score=5, rationale}
  WF->>SE: emit AttemptEvaluated (n=2, SUFFICIENT)
  WF->>SE: emit SessionAccepted (n=2)

  WF->>PRA: REWRITE_MEMORY(sessionId, report#2, SUFFICIENT, promptMemory)
  PRA-->>WF: MemoryDiff{2 block changes}
  WF->>SE: emit MemoryUpdated (diff)
  SE-->>SV: project
  SV-->>U: SSE update
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> RESEARCHING
  RESEARCHING --> EVALUATING: AttemptResearched
  EVALUATING --> RESEARCHING: EvaluatorVerdict = REFINE, attempts < max
  EVALUATING --> ACCEPTED: EvaluatorVerdict = SUFFICIENT
  EVALUATING --> MAX_ATTEMPTS_REACHED: EvaluatorVerdict = REFINE, attempts = max
  ACCEPTED --> MEMORY_REWRITING: MemoryUpdated emitted
  MAX_ATTEMPTS_REACHED --> MEMORY_REWRITING: MemoryUpdated emitted
  MEMORY_REWRITING --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ AttemptResearched : emits
  SessionEntity ||--o{ AttemptEvaluated : emits
  SessionEntity ||--o{ SessionAccepted : emits
  SessionEntity ||--o{ SessionMaxAttemptsReached : emits
  SessionEntity ||--o{ MemoryUpdated : emits
  SessionEntity ||--o{ QualityEvalRecorded : emits
  SessionEntity ||--o{ PromptDriftRecorded : emits
  SessionsView }o--|| SessionEntity : projects
  QueryQueue ||--o{ QuerySubmitted : emits
  QueryConsumer }o--|| QueryQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ResearchAgent` | `application/ResearchAgent.java` |
| `EvaluatorAgent` | `application/EvaluatorAgent.java` |
| `PromptRewriterAgent` | `application/PromptRewriterAgent.java` |
| `ResearchTasks` | `application/ResearchTasks.java` |
| `ResearchSessionWorkflow` | `application/ResearchSessionWorkflow.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `QueryQueue` | `application/QueryQueue.java` |
| `SessionsView` | `application/SessionsView.java` |
| `QueryConsumer` | `application/QueryConsumer.java` |
| `QuerySimulator` | `application/QuerySimulator.java` |
| `DriftSampler` | `application/DriftSampler.java` |
| `ResearchEndpoint` | `api/ResearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `researchStep`, `refineStep`, `evaluateStep`, and `rewriteMemoryStep` each carry `stepTimeout(Duration.ofSeconds(90))`. Research workloads can take substantially longer than the default 5-second timeout; the override is mandatory (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(maxAttemptsStep))` — any unrecoverable agent failure ends in `MAX_ATTEMPTS_REACHED`, not a hung workflow. The `rewriteMemoryStep` is outside the recovery scope; it runs after the terminal state is chosen.
- **Idempotency:** `ResearchEndpoint.submit` deduplicates on `(topic, requestedBy)` over a 15 s window. `DriftSampler` deduplicates on the `(oldFingerprint, newFingerprint)` pair so a tick that fires twice for the same transition is a no-op.
- **maxAttempts ceiling:** read from `self-improving-researcher.session.max-attempts` (default 3). The workflow checks the count BEFORE calling `refineStep`; it never recurses past the ceiling.
- **Memory store:** `PromptMemory` is a JSON file on the classpath (`governance/prompt-memory.json`). The `rewriteMemoryStep` writes the updated blocks back to this file. In a production deployment this would be an external KV store; the file-based approach is intentional for the out-of-the-box demo.
- **Drift fingerprint:** SHA-256 of the serialized `PromptMemory.blocks` JSON array, hex-encoded, first 16 chars. The `DriftSampler` stores the last-seen fingerprint in a local volatile field; a restart reseeds from the current file.
- **Rewrite step is unconditional:** the `PromptRewriterAgent` runs after every terminal transition — `ACCEPTED` and `MAX_ATTEMPTS_REACHED` alike. This ensures the memory always reflects the most recent session outcome, including failure patterns.
