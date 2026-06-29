# PLAN — self-correcting-rag

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

  Retriever[RetrieverAgent]:::agent
  Grader[RelevanceGraderAgent]:::agent
  Rewriter[QueryRewriterAgent]:::agent
  Generator[GeneratorAgent]:::agent
  HalluGrader[HallucinationGraderAgent]:::agent

  WF[RagWorkflow]:::wf
  Query[QueryEntity]:::ese
  Corpus[CorpusEntity]:::ese
  View[QueryView]:::view
  Consumer[QueryConsumer]:::cons
  Sim[QuerySimulator]:::ta
  Eval[EvalSampler]:::ta
  API[RagEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit query| Query
  Sim -.->|every 60s| API
  Query -.->|QueryCreated| Consumer
  Consumer -->|start workflow| WF
  WF -->|fetch docs| Corpus
  WF -->|retrieve| Retriever
  WF -->|grade relevance| Grader
  WF -->|rewrite query| Rewriter
  WF -->|generate answer| Generator
  WF -->|grade hallucination| HalluGrader
  WF -->|emit events| Query
  Query -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Query
```

## Interaction sequence — J1 (single-pass answer)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as RagEndpoint
  participant Qe as QueryEntity
  participant Qc as QueryConsumer
  participant W as RagWorkflow
  participant R as RetrieverAgent
  participant G as RelevanceGraderAgent
  participant Gen as GeneratorAgent
  participant H as HallucinationGraderAgent
  participant V as QueryView

  U->>API: POST /api/queries {queryText}
  API->>Qe: emit QueryCreated (RETRIEVING)
  API-->>U: 202 {queryId}
  Qe->>Qc: QueryCreated
  Qc->>W: start({queryId, queryText})

  W->>R: RETRIEVE(queryText, topK=5)
  R-->>W: RetrievalResult (5 docs)
  W->>Qe: emit DocumentsRetrieved
  Note over W: status GRADING
  W->>G: GRADE_RELEVANCE(query, docs)
  G-->>W: GradingResult (4 RELEVANT, 1 IRRELEVANT)
  W->>Qe: emit DocumentsGraded

  Note over W: noRelevantDocs? No → GENERATING
  W->>Gen: GENERATE(query, retainedDocs)
  Gen-->>W: GeneratedAnswer(answerText, citedDocIds)
  W->>Qe: emit AnswerGenerated

  Note over W: HALLUCINATION_CHECK
  W->>H: GRADE_HALLUCINATION(answer, retainedDocs)
  H-->>W: HallucinationVerdict{GROUNDED}
  W->>Qe: emit HallucinationChecked
  W->>Qe: emit QueryAnswered
  Qe-->>V: project
  V-->>U: SSE update
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> RETRIEVING
  RETRIEVING --> GRADING: DocumentsRetrieved
  GRADING --> GENERATING: relevant docs found
  GRADING --> REWRITING: no relevant docs, rewrites remaining
  GRADING --> FAILED_NO_RELEVANT_DOCS: no relevant docs, ceiling hit
  REWRITING --> RETRIEVING: QueryRewritten (retry retrieval)
  GENERATING --> HALLUCINATION_CHECK: AnswerGenerated
  HALLUCINATION_CHECK --> ANSWERED: GROUNDED
  HALLUCINATION_CHECK --> GENERATING: HALLUCINATED, generations remaining
  HALLUCINATION_CHECK --> FAILED_HALLUCINATION: HALLUCINATED, ceiling hit
  ANSWERED --> [*]
  FAILED_NO_RELEVANT_DOCS --> [*]
  FAILED_HALLUCINATION --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QueryCreated : emits
  QueryEntity ||--o{ RetrievalPassStarted : emits
  QueryEntity ||--o{ DocumentsRetrieved : emits
  QueryEntity ||--o{ DocumentsGraded : emits
  QueryEntity ||--o{ QueryRewritten : emits
  QueryEntity ||--o{ AnswerGenerated : emits
  QueryEntity ||--o{ HallucinationChecked : emits
  QueryEntity ||--o{ QueryAnswered : emits
  QueryEntity ||--o{ QueryFailedNoRelevantDocs : emits
  QueryEntity ||--o{ QueryFailedHallucination : emits
  QueryEntity ||--o{ EvalRecorded : emits
  QueryView }o--|| QueryEntity : projects
  CorpusEntity ||--o{ DocumentAdded : emits
  RagWorkflow }o--|| CorpusEntity : fetches
  QueryConsumer }o--|| QueryEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RetrieverAgent` | `application/RetrieverAgent.java` |
| `RelevanceGraderAgent` | `application/RelevanceGraderAgent.java` |
| `QueryRewriterAgent` | `application/QueryRewriterAgent.java` |
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `HallucinationGraderAgent` | `application/HallucinationGraderAgent.java` |
| `RagTasks` | `application/RagTasks.java` |
| `RagWorkflow` | `application/RagWorkflow.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `CorpusEntity` | `application/CorpusEntity.java` |
| `QueryView` | `application/QueryView.java` |
| `QueryConsumer` | `application/QueryConsumer.java` |
| `QuerySimulator` | `application/QuerySimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `RagEndpoint` | `api/RagEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `retrieveStep`, `gradeRelevanceStep`, `rewriteStep`, `generateStep`, and `hallucinationCheckStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failHallucinationStep))` — the workflow degrades to `FAILED_HALLUCINATION` on irrecoverable agent failure rather than hanging.
- **Rewrite ceiling:** read from `self-correcting-rag.retrieval.max-rewrite-attempts` (default 2). The workflow checks the count before scheduling a rewrite; it never recurses past the ceiling.
- **Generation ceiling:** read from `self-correcting-rag.generation.max-generation-attempts` (default 2). Same guard pattern.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(queryId, passNumber, stepKind)` so a tick that fires twice is a no-op on the entity side.
- **CorpusEntity consistency:** document fetches are served from the entity's in-memory state snapshot; no external vector store is required. The corpus is bootstrapped at startup from `src/main/resources/sample-events/corpus-documents.jsonl`.
- **Saga semantics:** both failure states (`FAILED_NO_RELEVANT_DOCS`, `FAILED_HALLUCINATION`) are graceful terminal states that preserve the full retrieval and generation history on the entity.
