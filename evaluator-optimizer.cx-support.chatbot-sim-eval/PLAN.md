# PLAN — chatbot-sim-eval

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

  SimUser[SimulatedUserAgent]:::agent
  Chatbot[ChatbotAgent]:::agent
  Evaluator[EvaluatorAgent]:::agent

  WF[SimulationWorkflow]:::wf
  Sim[SimulationEntity]:::ese
  Queue[ScenarioQueue]:::ese
  View[SimulationsView]:::view
  Consumer[ScenarioConsumer]:::cons
  ScenSim[ScenarioSimulator]:::ta
  EvalS[EvalSampler]:::ta
  API[SimulationEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue scenario| Queue
  ScenSim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|open + reply turns| SimUser
  WF -->|answer turns| Chatbot
  WF -->|evaluate transcript| Evaluator
  WF -->|emit events| Sim
  Sim -.->|projects| View
  API -->|query / SSE| View
  EvalS -.->|every 30s| Sim
```

## Interaction sequence — J1 (convergence: user signals resolution on turn 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SimulationEndpoint
  participant Q as ScenarioQueue
  participant C as ScenarioConsumer
  participant W as SimulationWorkflow
  participant SU as SimulatedUserAgent
  participant CB as ChatbotAgent
  participant EV as EvaluatorAgent
  participant E as SimulationEntity
  participant V as SimulationsView

  U->>API: POST /api/simulations {personaKey, issueDescription}
  API->>Q: append ScenarioSubmitted
  API-->>U: 202 {simulationId}
  Q->>C: ScenarioSubmitted
  C->>W: start({simulationId, personaKey, issueDescription, maxTurns=10})
  W->>E: emit SimulationCreated (RUNNING)

  W->>SU: OPEN_DIALOGUE(personaKey, issueDescription)
  SU-->>W: UserTurn #1 (signaledResolution=false)
  W->>E: emit UserTurnRecorded (n=1)
  W->>CB: ANSWER_TURN(userTurn #1)
  CB-->>W: AssistantTurn #1 (safeContentFlagged=false)
  Note over W: guardrailStep (safe-content check)
  W->>E: emit AssistantTurnRecorded (n=1, flagged=false)
  W->>SU: REPLY_AS_USER(personaKey, issueDescription, assistantTurn #1)
  SU-->>W: UserTurn #2 (signaledResolution=true)
  W->>E: emit UserTurnRecorded (n=2)

  W->>E: emit ConversationConcluded (RESOLVED) -> EVALUATING
  W->>EV: EVALUATE_TRANSCRIPT(turns[1..2])
  EV-->>W: EvalVerdict{PASS, score=9, summary}
  W->>E: emit EvalVerdictRecorded (PASS) -> PASSED_EVALUATION
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `SimulationEntity`

```mermaid
stateDiagram-v2
  [*] --> RUNNING
  RUNNING --> RUNNING: AssistantTurnRecorded + UserTurnRecorded (loop continues)
  RUNNING --> EVALUATING: ConversationConcluded (RESOLVED or MAX_TURNS_REACHED)
  EVALUATING --> PASSED_EVALUATION: EvalVerdictRecorded outcome=PASS
  EVALUATING --> FAILED_EVALUATION: EvalVerdictRecorded outcome=FAIL
  PASSED_EVALUATION --> [*]
  FAILED_EVALUATION --> [*]
```

## Entity model

```mermaid
erDiagram
  SimulationEntity ||--o{ SimulationCreated : emits
  SimulationEntity ||--o{ UserTurnRecorded : emits
  SimulationEntity ||--o{ AssistantTurnRecorded : emits
  SimulationEntity ||--o{ ConversationConcluded : emits
  SimulationEntity ||--o{ EvalVerdictRecorded : emits
  SimulationEntity ||--o{ EvalRecorded : emits
  SimulationsView }o--|| SimulationEntity : projects
  ScenarioQueue ||--o{ ScenarioSubmitted : emits
  ScenarioConsumer }o--|| ScenarioQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SimulatedUserAgent` | `application/SimulatedUserAgent.java` |
| `ChatbotAgent` | `application/ChatbotAgent.java` |
| `EvaluatorAgent` | `application/EvaluatorAgent.java` |
| `SimulationTasks` | `application/SimulationTasks.java` |
| `SimulationWorkflow` | `application/SimulationWorkflow.java` |
| `SimulationEntity` | `application/SimulationEntity.java` (state in `domain/Simulation.java`, events in `domain/SimulationEvent.java`) |
| `ScenarioQueue` | `application/ScenarioQueue.java` |
| `SimulationsView` | `application/SimulationsView.java` |
| `ScenarioConsumer` | `application/ScenarioConsumer.java` |
| `ScenarioSimulator` | `application/ScenarioSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `SimulationEndpoint` | `api/SimulationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `openDialogueStep`, `chatbotStep`, and `userReplyStep` each carry `stepTimeout(Duration.ofSeconds(90))`; `evaluateStep` carries `stepTimeout(Duration.ofSeconds(120))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — unrecoverable agent failure ends in `FAILED_EVALUATION`, not in a hung workflow.
- **Turn loop ceiling:** `maxTurns` is read from `chatbot-sim-eval.simulation.max-turns` (default 10). The workflow checks `turnCount < maxTurns` BEFORE scheduling the next chatbot step; it never recurses past the ceiling.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it scans the assistant turn text against a keyword/pattern set and sets `safeContentFlagged=true` if any pattern matches. Non-blocking: the turn is still committed.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `simulationId` so a tick that fires twice for the same completed simulation is a no-op on the entity side.
- **CI gate:** `GET /api/simulations/ci-gate?set=<ids>` is a synchronous read against `SimulationsView`; no side effects. Returns `{ passed: false }` if any id is not in a terminal state or is `FAILED_EVALUATION`.
- **Saga semantics:** there are no external side-effects to compensate. The `failStep` is the sole terminal state for both evaluation failure and irrecoverable agent error.
