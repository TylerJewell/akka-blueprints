# PLAN — multi-step-query-engine

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef eval fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[QueryPipelineWorkflow]:::wf
  Agent[QueryAgent]:::agent
  Decompose[DecomposeTools]:::tool
  Retrieve[RetrieveTools]:::tool
  Synthesize[SynthesizeTools]:::tool
  Evaluator[StoppingEvaluator]:::eval
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|decomposeStep runSingleTask| Agent
  Agent -->|invokes| Decompose
  Agent -->|invokes| Retrieve
  Agent -->|invokes| Synthesize
  Agent -->|DecomposedQuestion / EvidenceSet / QueryAnswer| WF
  WF -->|recordDecomposition/Evidence/Answer| Entity
  WF -->|evalStep score| Evaluator
  Evaluator -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant W as QueryPipelineWorkflow
  participant A as QueryAgent
  participant T as Tools (Decompose/Retrieve/Synthesize)
  participant Ev as StoppingEvaluator

  U->>API: POST /api/queries { question }
  API->>E: create(question)
  E-->>API: { queryId }
  API->>W: start(queryId, question)
  W->>E: startDecompose
  W->>A: runSingleTask(DECOMPOSE_QUESTION, question)
  A->>T: generateSubQuestions + rankSubQuestions
  T-->>A: List<SubQuestion>
  A-->>W: DecomposedQuestion
  W->>E: recordDecomposition
  W->>A: runSingleTask(RETRIEVE_EVIDENCE, subQuestions)
  A->>T: searchPassages (per sub-question) + fetchPassage
  T-->>A: List<Passage>
  A-->>W: EvidenceSet
  W->>E: recordEvidence
  W->>A: runSingleTask(SYNTHESIZE_ANSWER, evidenceSet)
  A->>T: draftSection (per sub-question) + composeAnswer
  T-->>A: List<AnswerSection> / QueryAnswer
  A-->>W: QueryAnswer
  W->>E: recordAnswer
  W->>Ev: score(queryAnswer, evidenceSet, decomposedQuestion)
  Ev-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `QueryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> DECOMPOSING: DecomposeStarted
  DECOMPOSING --> DECOMPOSED: QuestionDecomposed
  DECOMPOSED --> RETRIEVING: RetrieveStarted
  RETRIEVING --> RETRIEVED: EvidenceRetrieved
  RETRIEVED --> SYNTHESIZING: SynthesizeStarted
  SYNTHESIZING --> SYNTHESIZED: AnswerSynthesized
  SYNTHESIZED --> EVALUATED: AnswerEvaluated
  DECOMPOSING --> FAILED: QueryFailed
  RETRIEVING --> FAILED: QueryFailed
  SYNTHESIZING --> FAILED: QueryFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`AnswerEvaluated` is always recorded regardless of the score value — a score of 1 is still a valid terminal state. Only step timeout exhaustion or unrecoverable agent failure transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  QueryEntity ||--o{ QueryCreated : emits
  QueryEntity ||--o{ DecomposeStarted : emits
  QueryEntity ||--o{ QuestionDecomposed : emits
  QueryEntity ||--o{ RetrieveStarted : emits
  QueryEntity ||--o{ EvidenceRetrieved : emits
  QueryEntity ||--o{ SynthesizeStarted : emits
  QueryEntity ||--o{ AnswerSynthesized : emits
  QueryEntity ||--o{ AnswerEvaluated : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  QueryPipelineWorkflow }o--|| QueryEntity : reads-and-writes
  QueryAgent ||--o{ DecomposedQuestion : returns
  QueryAgent ||--o{ EvidenceSet : returns
  QueryAgent ||--o{ QueryAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/QueryRecord.java`, events in `domain/QueryEvent.java`) |
| `QueryPipelineWorkflow` | `application/QueryPipelineWorkflow.java` |
| `QueryAgent` | `application/QueryAgent.java` (tasks in `application/QueryTasks.java`) |
| `DecomposeTools` | `application/DecomposeTools.java` |
| `RetrieveTools` | `application/RetrieveTools.java` |
| `SynthesizeTools` | `application/SynthesizeTools.java` |
| `StoppingEvaluator` | `application/StoppingEvaluator.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `decomposeStep` 60 s, `retrieveStep` 60 s, `synthesizeStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + queryId` as the workflow id; restart of the same queryId is rejected by the workflow runtime. The agent instance id is `"agent-" + queryId` so each query has its own per-task conversation memory.
- **One agent per query**: `QueryAgent` runs three tasks per query — DECOMPOSE, RETRIEVE, SYNTHESIZE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the agent room to self-correct if a tool returns an empty result or an out-of-order call is made.
- **Eval is synchronous and deterministic**: `StoppingEvaluator` runs in-process inside `evalStep`. No LLM call, no external service — the same answer always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `decomposeStep` writes `QuestionDecomposed` BEFORE returning; `retrieveStep` reads the recorded `DecomposedQuestion` from the entity to build its task's instruction context; `synthesizeStep` reads both `DecomposedQuestion` and `EvidenceSet`. The agent itself is stateless across phases — it never holds decompose + retrieve + synthesize context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed query stays at the last successful event; the UI shows the partial state for the user.
