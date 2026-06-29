# PLAN — react-math-wiki

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef tool fill:#1a1a2e,stroke:#9b59b6,color:#9b59b6;

  API[ResearchEndpoint]:::ep
  Entity[ResearchEntity]:::ese
  WF[ResearchWorkflow]:::wf
  Agent[ResearchAgent]:::agent
  Guard[MathGuardrail]:::guard
  MathTool[MathEvaluator]:::tool
  WikiTool[WikipediaStub]:::tool
  Scorer[AnswerEvaluator]:::guard
  View[ResearchView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|runStep runSingleTask| Agent
  Agent -.->|before-tool-call evaluate_math| Guard
  Guard -->|allow / reject| Agent
  Agent -->|evaluate_math| MathTool
  Agent -->|search_wikipedia| WikiTool
  Agent -->|ResearchAnswer| WF
  WF -->|recordAnswer| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, numeric word-problem)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ResearchEndpoint
  participant E as ResearchEntity
  participant W as ResearchWorkflow
  participant A as ResearchAgent
  participant G as MathGuardrail
  participant M as MathEvaluator
  participant Ev as AnswerEvaluator

  U->>API: POST /api/questions
  API->>E: submit(question)
  E-->>API: { questionId }
  API->>W: start(questionId)
  W->>E: markRunning
  W->>A: runSingleTask(question.text)
  A->>G: before-tool-call(evaluate_math, "850 * 0.17")
  G-->>A: allow
  A->>M: evaluate_math("850 * 0.17")
  M-->>A: "144.5"
  A->>E: recordToolCall(step=1, EVALUATE_MATH, "850 * 0.17", "144.5")
  A-->>W: ResearchAnswer
  W->>E: recordAnswer(answer)
  W->>Ev: score(answer, question)
  Ev-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `ResearchEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RUNNING: RunStarted
  RUNNING --> RUNNING: ToolCallRecorded (loop)
  RUNNING --> ANSWERED: AnswerRecorded
  ANSWERED --> EVALUATED: EvaluationScored
  RUNNING --> FAILED: QuestionFailed (agent error)
  SUBMITTED --> FAILED: QuestionFailed (workflow start error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ResearchEntity ||--o{ QuestionSubmitted : emits
  ResearchEntity ||--o{ RunStarted : emits
  ResearchEntity ||--o{ ToolCallRecorded : emits
  ResearchEntity ||--o{ AnswerRecorded : emits
  ResearchEntity ||--o{ EvaluationScored : emits
  ResearchEntity ||--o{ QuestionFailed : emits
  ResearchView }o--|| ResearchEntity : projects
  ResearchWorkflow }o--|| ResearchEntity : reads-and-writes
  ResearchAgent ||--o{ ResearchAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ResearchEndpoint` | `api/ResearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ResearchEntity` | `application/ResearchEntity.java` (state in `domain/ResearchQuestion.java`, events in `domain/QuestionEvent.java`) |
| `ResearchWorkflow` | `application/ResearchWorkflow.java` |
| `ResearchAgent` | `application/ResearchAgent.java` (tasks in `application/ResearchTasks.java`) |
| `MathGuardrail` | `application/MathGuardrail.java` |
| `MathEvaluator` | `application/MathEvaluator.java` |
| `WikipediaStub` | `application/WikipediaStub.java` |
| `AnswerEvaluator` | `application/AnswerEvaluator.java` |
| `ResearchView` | `application/ResearchView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `runStep` 90 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ResearchWorkflow::error)`. The 90 s on `runStep` accommodates multi-step ReAct loops and LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"research-" + questionId` as the workflow id; the `ResearchEndpoint` only starts a workflow after the entity's `submit` command succeeds — a duplicate submit on the same `questionId` is rejected by the entity before the workflow start.
- **One agent per question**: the AutonomousAgent instance id is `"researcher-" + questionId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(8)` allows up to 8 ReAct steps.
- **Guardrail per tool call**: `MathGuardrail` fires once for every `evaluate_math` invocation within the ReAct loop, not once per response. A single task may trigger the guardrail multiple times if the agent calls `evaluate_math` multiple times. Each rejection counts as one iteration against the budget.
- **Eval is synchronous and deterministic**: `AnswerEvaluator` runs in-process inside `evalStep`. No LLM call — the same answer always scores the same. This is the single-agent invariant.
- **Tool calls are in-process**: both `MathEvaluator` and `WikipediaStub` execute inside the JVM with no network I/O. The blueprint runs entirely offline once the LLM call succeeds (or the mock is selected).
