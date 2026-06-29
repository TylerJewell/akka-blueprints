# PLAN — json-query-engine

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
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[JsonQueryWorkflow]:::wf
  Agent[QueryAgent]:::agent
  Parse[ParseTools]:::tool
  Traverse[TraverseTools]:::tool
  Respond[RespondTools]:::tool
  Guard[PathGuardrail]:::guard
  Scorer[AccuracyScorer]:::guard
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|parseStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Parse
  Agent -->|invokes| Traverse
  Agent -->|invokes| Respond
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|ParsedQuestion / TraversalResult / QueryResult| WF
  WF -->|recordParsed/Traversal/Result| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|AccuracyResult| WF
  WF -->|recordAccuracy| Entity
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
  participant W as JsonQueryWorkflow
  participant A as QueryAgent
  participant G as PathGuardrail
  participant T as Tools (Parse/Traverse/Respond)
  participant Sc as AccuracyScorer

  U->>API: POST /api/queries { question, docId }
  API->>E: create(question, docId)
  E-->>API: { queryId }
  API->>W: start(queryId, question, docId)
  W->>E: startParse
  W->>A: runSingleTask(PARSE_QUESTION, question + docId)
  A->>G: before-tool-call(identifyQueryIntent, PARSE)
  G-->>A: accept (status PARSING)
  A->>T: identifyQueryIntent + selectRootKeys
  T-->>A: QueryIntent + List<String>
  A-->>W: ParsedQuestion
  W->>E: recordParsed
  W->>A: runSingleTask(TRAVERSE_DOCUMENT, parsed)
  A->>G: before-tool-call(resolvePathExpression, TRAVERSE)
  G-->>A: accept (status TRAVERSING and parsed present — path syntax valid)
  A->>T: resolvePathExpression + extractValueAt
  T-->>A: List<PathMatch>
  A-->>W: TraversalResult
  W->>E: recordTraversal
  W->>A: runSingleTask(COMPOSE_RESPONSE, traversal)
  A->>G: before-tool-call(formatAnswer, RESPOND)
  G-->>A: accept (status RESPONDING and traversal present)
  A->>T: formatAnswer + buildCitations
  T-->>A: String + List<Citation>
  A-->>W: QueryResult
  W->>E: recordResult
  W->>Sc: score(result, traversal, parsed)
  Sc-->>W: AccuracyResult
  W->>E: recordAccuracy
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `QueryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PARSING: ParseStarted
  PARSING --> PARSED: QuestionParsed
  PARSED --> TRAVERSING: TraverseStarted
  TRAVERSING --> TRAVERSED: PathsTraversed
  TRAVERSED --> RESPONDING: RespondStarted
  RESPONDING --> RESPONDED: ResponseComposed
  RESPONDED --> EVALUATED: AccuracyScored
  PARSING --> FAILED: QueryFailed
  TRAVERSING --> FAILED: QueryFailed
  RESPONDING --> FAILED: QueryFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  QueryEntity ||--o{ QueryCreated : emits
  QueryEntity ||--o{ ParseStarted : emits
  QueryEntity ||--o{ QuestionParsed : emits
  QueryEntity ||--o{ TraverseStarted : emits
  QueryEntity ||--o{ PathsTraversed : emits
  QueryEntity ||--o{ RespondStarted : emits
  QueryEntity ||--o{ ResponseComposed : emits
  QueryEntity ||--o{ AccuracyScored : emits
  QueryEntity ||--o{ GuardrailRejected : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  JsonQueryWorkflow }o--|| QueryEntity : reads-and-writes
  QueryAgent ||--o{ ParsedQuestion : returns
  QueryAgent ||--o{ TraversalResult : returns
  QueryAgent ||--o{ QueryResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/QueryRecord.java`, events in `domain/QueryEvent.java`) |
| `JsonQueryWorkflow` | `application/JsonQueryWorkflow.java` |
| `QueryAgent` | `application/QueryAgent.java` (tasks in `application/QueryTasks.java`) |
| `ParseTools` | `application/ParseTools.java` |
| `TraverseTools` | `application/TraverseTools.java` |
| `RespondTools` | `application/RespondTools.java` |
| `PathGuardrail` | `application/PathGuardrail.java` |
| `AccuracyScorer` | `application/AccuracyScorer.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `parseStep` 60 s, `traverseStep` 60 s, `respondStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(JsonQueryWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"workflow-" + queryId` as the workflow id; restart of the same queryId is rejected by the workflow runtime. The agent instance id is `"agent-" + queryId` so each query has its own per-task conversation memory.
- **One agent per query**: `QueryAgent` runs three tasks per query — PARSE, TRAVERSE, RESPOND — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the path guardrail room to reject a malformed expression and still let the agent self-correct.
- **Guardrail-driven retry**: when `PathGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `AccuracyScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same query result always scores the same.
- **Task-boundary handoff is the dependency contract**: `parseStep` writes `QuestionParsed` BEFORE returning; `traverseStep` reads the recorded `ParsedQuestion` from the entity to build its task's instruction context; `respondStep` reads both `ParsedQuestion` and `TraversalResult`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed query stays at the last successful event; the UI shows the partial state for the user.
