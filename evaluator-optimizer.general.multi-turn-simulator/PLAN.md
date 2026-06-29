# PLAN — multi-turn-simulator

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

  Actor[ActorAgent]:::agent
  Eval[EvaluatorAgent]:::agent

  WF[SimulationWorkflow]:::wf
  Sess[SessionEntity]:::ese
  Queue[ScenarioQueue]:::ese
  View[SessionsView]:::view
  Consumer[ScenarioConsumer]:::cons
  Sim[ScenarioSimulator]:::ta
  Drift[DriftSampler]:::ta
  API[SimulationEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue scenario| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|next utterance| Actor
  WF -->|score turn / close| Eval
  WF -->|emit events| Sess
  Sess -.->|projects| View
  API -->|query / SSE| View
  Drift -.->|every 45s| Sess
```

## Interaction sequence — J1 (clean session, 2 turns)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SimulationEndpoint
  participant Q as ScenarioQueue
  participant C as ScenarioConsumer
  participant W as SimulationWorkflow
  participant A as ActorAgent
  participant T as TargetAgentProxy
  participant EV as EvaluatorAgent
  participant S as SessionEntity
  participant V as SessionsView

  U->>API: POST /api/sessions {persona, goal}
  API->>Q: append ScenarioSubmitted
  API-->>U: 202 {sessionId}
  Q->>C: ScenarioSubmitted
  C->>W: start({sessionId, persona, goal, maxTurns=10})
  W->>S: emit SessionCreated (RUNNING)

  W->>A: INITIATE_TURN(persona, goal)
  A-->>W: ActorUtterance turn=1
  W->>S: emit TurnUtteranceRecorded (n=1)
  W->>T: forward utterance
  T-->>W: TargetResponse turn=1
  W->>S: emit TurnResponseRecorded (n=1)
  Note over W: guardrailStep (deterministic keyword check)
  W->>S: emit TurnGuardrailVerdictRecorded (passed=true)
  W->>EV: SCORE_TURN(utterance, response)
  EV-->>W: TurnVerdict{PASS, score=4, driftFlagged=false}
  W->>S: emit TurnVerdictRecorded (n=1, PASS)

  W->>A: CONTINUE_TURN(persona, goal, history)
  A-->>W: ActorUtterance turn=2 (goal-complete signal)
  W->>S: emit TurnUtteranceRecorded (n=2)
  W->>T: forward utterance
  T-->>W: TargetResponse turn=2
  W->>S: emit TurnGuardrailVerdictRecorded (passed=true)
  W->>EV: SCORE_TURN(utterance, response)
  EV-->>W: TurnVerdict{PASS, score=5, driftFlagged=false}
  W->>S: emit TurnVerdictRecorded (n=2, PASS)
  W->>EV: CLOSE_SESSION(turns)
  EV-->>W: SessionVerdict{CLEAN, sessionScore=4, driftFlagged=false}
  W->>S: emit SessionClosed (COMPLETED)
  S-->>V: project
  V-->>U: SSE update
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> RUNNING
  RUNNING --> RUNNING: TurnVerdictRecorded PASS or REVISE, turns < max
  RUNNING --> RUNNING: guardrail blocked, syntheticVerdictRecorded
  RUNNING --> COMPLETED: SessionClosed outcome=CLEAN
  RUNNING --> FLAGGED: SessionClosed outcome=FLAGGED
  RUNNING --> HALTED: turns = maxTurns or step recovery failover
  COMPLETED --> [*]
  FLAGGED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ TurnUtteranceRecorded : emits
  SessionEntity ||--o{ TurnResponseRecorded : emits
  SessionEntity ||--o{ TurnGuardrailVerdictRecorded : emits
  SessionEntity ||--o{ TurnVerdictRecorded : emits
  SessionEntity ||--o{ SessionClosed : emits
  SessionEntity ||--o{ DriftEvalRecorded : emits
  SessionsView }o--|| SessionEntity : projects
  ScenarioQueue ||--o{ ScenarioSubmitted : emits
  ScenarioConsumer }o--|| ScenarioQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ActorAgent` | `application/ActorAgent.java` |
| `EvaluatorAgent` | `application/EvaluatorAgent.java` |
| `SimulationTasks` | `application/SimulationTasks.java` |
| `SimulationWorkflow` | `application/SimulationWorkflow.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `ScenarioQueue` | `application/ScenarioQueue.java` |
| `SessionsView` | `application/SessionsView.java` |
| `ScenarioConsumer` | `application/ScenarioConsumer.java` |
| `ScenarioSimulator` | `application/ScenarioSimulator.java` |
| `DriftSampler` | `application/DriftSampler.java` |
| `SimulationEndpoint` | `api/SimulationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TargetAgentProxy` | `application/TargetAgentProxy.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `actorStep`, `targetStep`, and `scoreStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling or network-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(haltStep))` — the workflow degrades to `HALTED` on irrecoverable agent or network failure rather than hanging.
- **Idempotency:** `SimulationEndpoint.submit` uses `(persona, goal, submittedBy)` over a 10 s window as the dedup key.
- **DriftSampler idempotency:** the sampler keys its `recordDriftEval` calls on `(sessionId, turnNumber)` so a tick that fires twice for the same turn is a no-op on the entity side.
- **maxTurns ceiling:** read from `multi-turn-simulator.simulation.max-turns` (default 10). The workflow checks the count BEFORE calling `actorStep` for the next turn; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate for. The halt mechanism (`HT1`) is the only "compensation"; it preserves every turn record and every verdict on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it scans the raw response text for sensitive-pattern keywords and either advances to `scoreStep` or synthesizes a `TurnVerdict` with `outcome = POLICY_VIOLATION`. The synthetic verdict is deterministic; it never calls the EvaluatorAgent.
- **TargetAgentProxy:** by default, proxies to a configurable URL (`multi-turn-simulator.simulation.target-url`). When the value is `"stub"`, the proxy returns a deterministic canned response keyed on `(persona, turnNumber)` from `src/main/resources/stub-responses/target.json`. This lets the service run out of the box with no external endpoint.
