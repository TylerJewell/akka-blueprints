# PLAN — memory-eval-loop

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

  Answer[AnswerAgent]:::agent
  Scorer[ScorerAgent]:::agent

  WF[AnswerWorkflow]:::wf
  Session[SessionEntity]:::ese
  Memory[MemoryStore]:::ese
  SView[SessionView]:::view
  MView[MemoryView]:::view
  Consumer[QuestionConsumer]:::cons
  Sim[QuestionSimulator]:::ta
  Eval[EvalSampler]:::ta
  Drift[DriftWatcher]:::ta
  API[SessionEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|create session| Session
  Sim -.->|every 60s| API
  Session -.->|SessionCreated| Consumer
  Consumer -->|start workflow| WF
  WF -->|retrieve| MView
  WF -->|answer / revise| Answer
  WF -->|score| Scorer
  WF -->|emit events| Session
  WF -->|addEntry post-turn| Memory
  Session -.->|projects| SView
  Memory -.->|projects| MView
  API -->|query / SSE| SView
  API -->|query| MView
  Eval -.->|every 30s| Session
  Drift -.->|every 10m| Session
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SessionEndpoint
  participant S as SessionEntity
  participant C as QuestionConsumer
  participant W as AnswerWorkflow
  participant MV as MemoryView
  participant A as AnswerAgent
  participant SC as ScorerAgent
  participant MS as MemoryStore
  participant SV as SessionView

  U->>API: POST /api/sessions {text, userId}
  API->>S: emit SessionCreated (ANSWERING)
  API-->>U: 202 {sessionId}
  S->>C: SessionCreated
  C->>W: start({sessionId, userId, questionText, maxAttempts=3})

  W->>MV: getEntriesForUser(userId, topK=10)
  MV-->>W: List<MemoryEntry> (retrieved context)
  W->>A: ANSWER(questionText, retrievedEntries)
  A-->>W: AnswerAttempt #1 (3 cited entries)
  W->>S: emit AnswerAttempted (n=1)
  W->>S: status SCORING
  W->>SC: SCORE_ANSWER(AnswerAttempt #1)
  SC-->>W: Score{IMPROVE, score=3, 3 bullets}
  W->>S: emit AttemptScored (n=1, IMPROVE)

  W->>A: REVISE_ANSWER(questionText, priorAnswer, retrievedEntries, notes)
  A-->>W: AnswerAttempt #2 (revised, 3 cited entries)
  W->>S: emit AnswerAttempted (n=2)
  W->>SC: SCORE_ANSWER(AnswerAttempt #2)
  SC-->>W: Score{PASS, score=5, rationale}
  W->>S: emit AttemptScored (n=2, PASS)
  W->>S: emit SessionAccepted (n=2)
  W->>MS: addEntry(sanitized fact fragments)
  MS-->>MV: project
  S-->>SV: project
  SV-->>U: SSE update
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> ANSWERING
  ANSWERING --> SCORING: AnswerAttempted
  SCORING --> ANSWERING: Score = IMPROVE, attempts < max
  SCORING --> ACCEPTED: Score = PASS
  SCORING --> REJECTED_FINAL: Score = IMPROVE, attempts = max
  ACCEPTED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ AnswerAttempted : emits
  SessionEntity ||--o{ AttemptScored : emits
  SessionEntity ||--o{ SessionAccepted : emits
  SessionEntity ||--o{ SessionRejectedFinal : emits
  SessionEntity ||--o{ EvalRecorded : emits
  SessionEntity ||--o{ DriftCheckRecorded : emits
  SessionView }o--|| SessionEntity : projects
  MemoryStore ||--o{ MemoryEntryAdded : emits
  MemoryStore ||--o{ MemoryEntryPiiRedacted : emits
  MemoryView }o--|| MemoryStore : projects
  QuestionConsumer }o--|| SessionEntity : subscribes
  AnswerWorkflow }o--|| MemoryView : retrieves
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AnswerAgent` | `application/AnswerAgent.java` |
| `ScorerAgent` | `application/ScorerAgent.java` |
| `EvalTasks` | `application/EvalTasks.java` |
| `AnswerWorkflow` | `application/AnswerWorkflow.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `MemoryStore` | `application/MemoryStore.java` (state in `domain/MemoryState.java`, events in `domain/MemoryEvent.java`) |
| `SessionView` | `application/SessionView.java` |
| `MemoryView` | `application/MemoryView.java` |
| `QuestionConsumer` | `application/QuestionConsumer.java` |
| `QuestionSimulator` | `application/QuestionSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `DriftWatcher` | `application/DriftWatcher.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `answerStep` and `scoreStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Memory write ordering:** `memoryWriteStep` runs only after the session is in a terminal state. The PII sanitizer runs inline in the step before the `MemoryStore.addEntry` call; no unsanitized content reaches the entity command handler.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(sessionId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **DriftWatcher sentinel:** drift check state is written to a fixed entity id (`"drift-watch-singleton"`) rather than creating one entity per check. The TimedAction reads `SessionView` (no side-effect) and writes exactly one event.
- **maxAttempts ceiling:** read from `evals-with-memory.answer.max-attempts` (default 3). The workflow checks the count BEFORE calling `answerStep` for the next iteration.
- **Saga semantics:** there is no external side-effect to compensate other than the memory write. If `memoryWriteStep` fails, the session is already in `ACCEPTED`; the failure is logged and the step is retried up to 2 times before surfacing a warning. Memory writes are append-only and idempotent on `(sourceSessionId, content-hash)`.
- **MemoryView retrieval:** `getEntriesForUser` is called at the start of each answer attempt (not just the first), so if memory entries were written by a prior session, the revised answer can cite them.
