# PLAN — lats-tree-search

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

  Search[SearchAgent]:::agent
  Reflect[ReflectorAgent]:::agent

  WF[TreeSearchWorkflow]:::wf
  Tree[SearchTreeEntity]:::ese
  Queue[ProblemQueue]:::ese
  View[TreeView]:::view
  Consumer[ProblemConsumer]:::cons
  Sim[ProblemSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[SearchEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit problem| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|expand node| Search
  WF -->|reflect candidate| Reflect
  WF -->|emit events| Tree
  Tree -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 45s| Tree
```

## Interaction sequence — J1 (solved on second expansion)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SearchEndpoint
  participant Q as ProblemQueue
  participant C as ProblemConsumer
  participant W as TreeSearchWorkflow
  participant S as SearchAgent
  participant R as ReflectorAgent
  participant T as SearchTreeEntity
  participant V as TreeView

  U->>API: POST /api/trees {taskDescription, nodeBudget}
  API->>Q: append ProblemSubmitted
  API-->>U: 202 {treeId}
  Q->>C: ProblemSubmitted
  C->>W: start({treeId, taskDescription, nodeBudget=20})
  W->>T: emit TreeCreated (EXPANDING)

  W->>S: EXPAND_NODE(taskDescription, root, depth=0)
  S-->>W: NodeExpansion{3 candidates}
  W->>T: emit NodeExpanded (3 candidates)
  W->>R: REFLECT_NODE(candidate-A)
  R-->>W: NodeScore{score=5, isTerminal=false}
  W->>R: REFLECT_NODE(candidate-B)
  R-->>W: NodeScore{score=3, isTerminal=false}
  W->>R: REFLECT_NODE(candidate-C)
  R-->>W: NodeScore{score=7, isTerminal=false}
  W->>T: emit NodeReflected (×3)
  W->>T: emit BestPathAdvanced (selected=candidate-C)
  W->>T: emit BackpropRecorded (siblings A, B get delta)
  Note over W: status → REFLECTING then EXPANDING

  W->>S: EXPAND_NODE(taskDescription, candidate-C, depth=1)
  S-->>W: NodeExpansion{3 candidates}
  W->>T: emit NodeExpanded
  W->>R: REFLECT_NODE(candidate-D)
  R-->>W: NodeScore{score=9, isTerminal=true}
  W->>T: emit NodeReflected (candidate-D terminal)
  W->>T: emit TreeSolved (bestPath=[root,C,D])
  T-->>V: project
  V-->>U: SSE update
```

## State machine — `SearchTreeEntity`

```mermaid
stateDiagram-v2
  [*] --> EXPANDING
  EXPANDING --> REFLECTING: NodeExpanded emitted
  REFLECTING --> EXPANDING: BestPathAdvanced, not terminal, budget remains
  REFLECTING --> SOLVED: isTerminal = true
  REFLECTING --> BUDGET_EXHAUSTED: budget reached, no terminal
  SOLVED --> [*]
  BUDGET_EXHAUSTED --> [*]
```

## Entity model

```mermaid
erDiagram
  SearchTreeEntity ||--o{ TreeCreated : emits
  SearchTreeEntity ||--o{ NodeExpanded : emits
  SearchTreeEntity ||--o{ NodeReflected : emits
  SearchTreeEntity ||--o{ BestPathAdvanced : emits
  SearchTreeEntity ||--o{ BackpropRecorded : emits
  SearchTreeEntity ||--o{ TreeSolved : emits
  SearchTreeEntity ||--o{ BudgetExhausted : emits
  SearchTreeEntity ||--o{ EvalRecorded : emits
  TreeView }o--|| SearchTreeEntity : projects
  ProblemQueue ||--o{ ProblemSubmitted : emits
  ProblemConsumer }o--|| ProblemQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SearchAgent` | `application/SearchAgent.java` |
| `ReflectorAgent` | `application/ReflectorAgent.java` |
| `SearchTasks` | `application/SearchTasks.java` |
| `TreeSearchWorkflow` | `application/TreeSearchWorkflow.java` |
| `SearchTreeEntity` | `application/SearchTreeEntity.java` (state in `domain/SearchTree.java`, events in `domain/TreeEvent.java`) |
| `ProblemQueue` | `application/ProblemQueue.java` |
| `TreeView` | `application/TreeView.java` |
| `ProblemConsumer` | `application/ProblemConsumer.java` |
| `ProblemSimulator` | `application/ProblemSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `SearchEndpoint` | `api/SearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `expandStep` and `reflectStep` each carry `stepTimeout(Duration.ofSeconds(90))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep))` — the workflow degrades to `BUDGET_EXHAUSTED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `SearchEndpoint.submit` uses `(taskDescription, submittedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(treeId, nodeId)` so a tick that fires twice for the same node is a no-op on the entity side.
- **nodeBudget ceiling:** read from `lats-tree-search.search.node-budget` (default 20). The workflow checks the count BEFORE calling `expandStep` for the next iteration; it never recurses past the ceiling.
- **Reflect loop:** `reflectStep` calls `ReflectorAgent` once per candidate in the current expansion batch. Each call is an independent `runSingleTask(REFLECT_NODE)` with its own `stepTimeout(90s)`.
- **Backpropagation:** after `selectStep` picks the highest-scoring candidate, `BackpropRecorded` events are emitted for every non-selected sibling, each carrying the winning node's score × 0.1 as the delta. The entity applies these as a view-side annotation; they do not alter the tree's structural state.
- **Score threshold:** `lats-tree-search.search.score-threshold` (default 8). If the highest candidate score in a reflection batch is below this threshold, `selectStep` still advances the best path but logs a warning; it does not halt early. Early halt on sub-threshold would require a second halt control.
