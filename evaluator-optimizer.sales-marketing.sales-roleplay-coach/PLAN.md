# PLAN — sales-roleplay-coach

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

  Buyer[BuyerSimulatorAgent]:::agent
  Coach[CoachAgent]:::agent

  WF[SessionWorkflow]:::wf
  Sess[SessionEntity]:::ese
  Queue[ScenarioQueue]:::ese
  View[SessionsView]:::view
  Consumer[ScenarioRequestConsumer]:::cons
  Sim[ScenarioSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[CoachEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue scenario| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|open session / respond| Buyer
  WF -->|score turn| Coach
  WF -->|emit events| Sess
  Sess -.->|projects| View
  API -->|query / SSE| View
  API -->|rep turn submission| WF
  Eval -.->|every 30s| Sess
```

## Interaction sequence — J1 (convergence on turn 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as Rep (User)
  participant API as CoachEndpoint
  participant Q as ScenarioQueue
  participant C as ScenarioRequestConsumer
  participant W as SessionWorkflow
  participant B as BuyerSimulatorAgent
  participant K as CoachAgent
  participant E as SessionEntity
  participant V as SessionsView

  U->>API: POST /api/sessions {buyerPersona, product, dealStage}
  API->>Q: append ScenarioSubmitted
  API-->>U: 202 {sessionId}
  Q->>C: ScenarioSubmitted
  C->>W: start({sessionId, dealContext, maxTurns=5})
  W->>E: emit SessionCreated (PITCHING)

  W->>B: OPEN_SESSION(dealContext)
  B-->>W: BuyerTurn{opener, signal=SKEPTICAL}
  W->>E: emit BuyerResponseRecorded (turn 0 opener)

  U->>API: POST /api/sessions/{id}/turns {text: "Our platform..."}
  API->>W: deliver RepTurn
  W->>E: emit RepTurnRecorded (turn 1)
  Note over W: guardrailStep (deterministic content check)
  W->>E: emit TurnGuardrailVerdictRecorded (passed=true)
  W->>E: status EVALUATING
  W->>B: RESPOND_TO_PITCH(dealContext, repTurn#1)
  B-->>W: BuyerTurn{objection, signal=RESISTANT}
  W->>E: emit BuyerResponseRecorded (turn 1)
  W->>K: SCORE_TURN(dealContext, repTurn#1, buyerResponse#1)
  K-->>W: CoachVerdict{REVISE, score=3, 3 bullets}
  W->>E: emit TurnCoachVerdicted (turn 1, REVISE)

  U->>API: POST /api/sessions/{id}/turns {text: "Let me address that..."}
  API->>W: deliver RepTurn
  W->>E: emit RepTurnRecorded (turn 2)
  W->>E: emit TurnGuardrailVerdictRecorded (passed=true)
  W->>B: RESPOND_TO_PITCH(dealContext, repTurn#2, coachNotes)
  B-->>W: BuyerTurn{positive, signal=INTERESTED}
  W->>K: SCORE_TURN(dealContext, repTurn#2, buyerResponse#2)
  K-->>W: CoachVerdict{ACCEPT, score=5, rationale}
  W->>E: emit TurnCoachVerdicted (turn 2, ACCEPT)
  W->>E: emit SessionPassed (turn 2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> PITCHING
  PITCHING --> EVALUATING: RepTurn recorded + guardrail passed
  PITCHING --> PITCHING: guardrail blocked, re-pitch
  EVALUATING --> PITCHING: CoachVerdict = REVISE, turns < max
  EVALUATING --> PASSED: CoachVerdict = ACCEPT
  EVALUATING --> EXHAUSTED: CoachVerdict = REVISE, turns = max
  PASSED --> [*]
  EXHAUSTED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ RepTurnRecorded : emits
  SessionEntity ||--o{ TurnGuardrailVerdictRecorded : emits
  SessionEntity ||--o{ BuyerResponseRecorded : emits
  SessionEntity ||--o{ TurnCoachVerdicted : emits
  SessionEntity ||--o{ SessionPassed : emits
  SessionEntity ||--o{ SessionExhausted : emits
  SessionEntity ||--o{ EvalRecorded : emits
  SessionsView }o--|| SessionEntity : projects
  ScenarioQueue ||--o{ ScenarioSubmitted : emits
  ScenarioRequestConsumer }o--|| ScenarioQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BuyerSimulatorAgent` | `application/BuyerSimulatorAgent.java` |
| `CoachAgent` | `application/CoachAgent.java` |
| `SalesCoachTasks` | `application/SalesCoachTasks.java` |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `ScenarioQueue` | `application/ScenarioQueue.java` |
| `SessionsView` | `application/SessionsView.java` |
| `ScenarioRequestConsumer` | `application/ScenarioRequestConsumer.java` |
| `ScenarioSimulator` | `application/ScenarioSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `CoachEndpoint` | `api/CoachEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `openBuyerStep`, `buyerResponseStep`, and `coachStep` each carry `stepTimeout(Duration.ofSeconds(90))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep))` — the workflow degrades to `EXHAUSTED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `CoachEndpoint.createSession` uses `(buyerPersona, product, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(sessionId, turnNumber)` so a tick that fires twice for the same turn is a no-op on the entity side.
- **maxTurns ceiling:** read from `sales-coach.session.max-turns` (default 5). The workflow checks the count BEFORE waiting for the next rep turn; it never accepts a submission past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism is the only "compensation"; it preserves the best turn and every coaching verdict on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it checks the rep's turn text against the prohibited-content list (loaded from `src/main/resources/prohibited-patterns.txt`) and either advances to `buyerResponseStep` or returns to the repTurnStep wait state with a structured feedback note. The structured feedback never becomes an LLM-generated critique; it stays a deterministic `CoachingNotes` payload with a single bullet.
- **Rep turn delivery:** the workflow pauses at `repTurnStep` waiting for the rep to POST to `/api/sessions/{id}/turns`. `CoachEndpoint` delivers the turn into the workflow via a `ComponentClient` call. If no turn arrives within the step timeout, the workflow fails over to `exhaustStep`.
