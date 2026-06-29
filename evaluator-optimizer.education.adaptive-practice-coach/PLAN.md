# PLAN — adaptive-practice-coach

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

  Gen[ItemGeneratorAgent]:::agent
  Scorer[ScorerAgent]:::agent

  WF[PracticeSessionWorkflow]:::wf
  Session[LearnerSessionEntity]:::ese
  Queue[EnrollmentQueue]:::ese
  View[SessionsView]:::view
  Consumer[SessionRequestConsumer]:::cons
  Sim[EnrollmentSimulator]:::ta
  Diag[DiagnosticSampler]:::ta
  API[CoachEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue enrollment| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise item| Gen
  WF -->|score response| Scorer
  WF -->|emit events| Session
  Session -.->|projects| View
  API -->|query / SSE| View
  API -->|submit learner response| WF
  Diag -.->|every 30s| Session
```

## Interaction sequence — J1 (mastery on item 3)

```mermaid
sequenceDiagram
  autonumber
  participant U as Learner
  participant API as CoachEndpoint
  participant Q as EnrollmentQueue
  participant C as SessionRequestConsumer
  participant W as PracticeSessionWorkflow
  participant G as ItemGeneratorAgent
  participant S as ScorerAgent
  participant E as LearnerSessionEntity
  participant V as SessionsView

  U->>API: POST /api/sessions {topic, learnerId}
  API->>Q: append EnrollmentRequested
  API-->>U: 202 {sessionId}
  Q->>C: EnrollmentRequested
  C->>W: start({sessionId, topic, learnerId, maxItems=10})
  W->>E: emit SessionStarted (ACTIVE)

  W->>G: GENERATE_ITEM(topic, difficulty=3)
  G-->>W: PracticeItem #1
  Note over W: curriculumStep (deterministic alignment check)
  W->>E: emit ItemCurriculumVerdictRecorded (passed=true)
  W->>E: status AWAITING_RESPONSE
  API-->>U: SSE item #1 question
  U->>API: POST /api/sessions/{id}/response {response}
  W->>S: SCORE_RESPONSE(item #1, response)
  S-->>W: ScoreReport{CONTINUE, delta=+0.10}
  W->>E: emit ItemScored (n=1, CONTINUE)

  W->>G: GENERATE_ITEM(topic, difficulty=4, diagnosticTag)
  G-->>W: PracticeItem #2
  W->>E: emit ItemCurriculumVerdictRecorded (passed=true)
  W->>E: status AWAITING_RESPONSE
  U->>API: POST /api/sessions/{id}/response {response}
  W->>S: SCORE_RESPONSE(item #2, response)
  S-->>W: ScoreReport{MASTERED, delta=+0.25}
  W->>E: emit ItemScored (n=2, MASTERED)
  W->>E: emit SessionMastered (masteryEstimate=0.85)
  E-->>V: project
  V-->>U: SSE update MASTERED
```

## State machine — `LearnerSessionEntity`

```mermaid
stateDiagram-v2
  [*] --> ACTIVE
  ACTIVE --> AWAITING_RESPONSE: ItemGenerated + curriculum passed
  ACTIVE --> ACTIVE: curriculum blocked, re-generate
  AWAITING_RESPONSE --> ACTIVE: ScoreReport = CONTINUE, items < max
  AWAITING_RESPONSE --> MASTERED: ScoreReport = MASTERED
  AWAITING_RESPONSE --> CAP_REACHED: ScoreReport = CONTINUE, items = max
  MASTERED --> [*]
  CAP_REACHED --> [*]
```

## Entity model

```mermaid
erDiagram
  LearnerSessionEntity ||--o{ SessionStarted : emits
  LearnerSessionEntity ||--o{ ItemGenerated : emits
  LearnerSessionEntity ||--o{ ItemCurriculumVerdictRecorded : emits
  LearnerSessionEntity ||--o{ ResponseSubmitted : emits
  LearnerSessionEntity ||--o{ ItemScored : emits
  LearnerSessionEntity ||--o{ SessionMastered : emits
  LearnerSessionEntity ||--o{ SessionCapReached : emits
  LearnerSessionEntity ||--o{ DiagnosticRecorded : emits
  SessionsView }o--|| LearnerSessionEntity : projects
  EnrollmentQueue ||--o{ EnrollmentRequested : emits
  SessionRequestConsumer }o--|| EnrollmentQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ItemGeneratorAgent` | `application/ItemGeneratorAgent.java` |
| `ScorerAgent` | `application/ScorerAgent.java` |
| `CoachTasks` | `application/CoachTasks.java` |
| `PracticeSessionWorkflow` | `application/PracticeSessionWorkflow.java` |
| `LearnerSessionEntity` | `application/LearnerSessionEntity.java` (state in `domain/LearnerSession.java`, events in `domain/SessionEvent.java`) |
| `EnrollmentQueue` | `application/EnrollmentQueue.java` |
| `SessionsView` | `application/SessionsView.java` |
| `SessionRequestConsumer` | `application/SessionRequestConsumer.java` |
| `EnrollmentSimulator` | `application/EnrollmentSimulator.java` |
| `DiagnosticSampler` | `application/DiagnosticSampler.java` |
| `CoachEndpoint` | `api/CoachEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `scoreStep` each carry `stepTimeout(Duration.ofSeconds(60))`. `awaitResponseStep` carries `stepTimeout(Duration.ofSeconds(300))` — the learner has 5 minutes to respond before the step re-parks itself. The default 5-second timeout never applies to agent-calling or wait steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(capStep))` — the workflow ends with `CAP_REACHED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `CoachEndpoint.startSession` uses `(topic, learnerId)` over a 10 s window as the dedup key. `DiagnosticSampler` keys its `recordDiagnostic` calls on `(sessionId, itemNumber)` so a tick that fires twice for the same item is a no-op on the entity.
- **maxItems ceiling:** read from `adaptive-practice-coach.session.max-items` (default 10). The workflow checks the count BEFORE generating the next item; it never recurses past the ceiling.
- **Mastery threshold:** read from `adaptive-practice-coach.session.mastery-threshold` (default 0.8). The workflow compares `masteryEstimate` after each `ScoreReport`; on first crossing, it transitions to `masterStep` regardless of how many items remain.
- **Difficulty adaptation:** after each scored item, the workflow reads `ScoreReport.correct` and adjusts `currentDifficulty`: correct → increment by 1 (capped at 5); incorrect → decrement by 1 (floor at 1). The next `generateStep` receives the updated difficulty so the loop calibrates to the learner's demonstrated level.
- **Curriculum step:** `curriculumStep` is pure-function (no LLM call); it checks that `item.topic` textually matches the session topic and that `question` and `answerKey` are non-empty strings. On FAIL, emits `ItemCurriculumVerdictRecorded` with `passed = false` and transitions back to `generateStep` with a fixed feedback note. The replacement item still counts toward `maxItems`.
