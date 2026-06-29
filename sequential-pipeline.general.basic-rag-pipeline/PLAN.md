# PLAN — basic-rag-pipeline

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
  classDef store fill:#0e1a2a,stroke:#60a5fa,color:#60a5fa;

  API[RagEndpoint]:::ep
  Entity[RagSessionEntity]:::ese
  WF[RagPipelineWorkflow]:::wf
  Agent[RagAgent]:::agent
  Ingest[IngestTools]:::tool
  Query[QueryTools]:::tool
  VS[VectorStore]:::store
  Guard[AnswerGuardrail]:::guard
  View[RagSessionView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|ingestStep runSingleTask| Agent
  Agent -->|invokes| Ingest
  Ingest -->|writes chunks| VS
  WF -->|queryStep runSingleTask| Agent
  Agent -->|invokes| Query
  Query -->|reads chunks| VS
  Agent -.->|before-agent-response| Guard
  Guard -->|recordGuardrailOutcome| Entity
  Agent -->|IndexedCorpus / RagAnswer| WF
  WF -->|recordCorpus/Answer| Entity
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
  participant API as RagEndpoint
  participant E as RagSessionEntity
  participant W as RagPipelineWorkflow
  participant A as RagAgent
  participant G as AnswerGuardrail
  participant IT as IngestTools
  participant QT as QueryTools
  participant VS as VectorStore

  U->>API: POST /api/sessions { question }
  API->>E: create(question)
  E-->>API: { sessionId }
  API->>W: start(sessionId, question)
  W->>E: startIngest
  W->>A: runSingleTask(INGEST_CORPUS)
  A->>IT: loadDocuments("default")
  IT-->>A: List<DocumentSource>
  A->>IT: indexChunks(documents)
  IT->>VS: store chunks
  VS-->>IT: List<IndexedChunk>
  IT-->>A: List<IndexedChunk>
  A-->>W: IndexedCorpus
  W->>E: recordCorpus
  W->>E: startQuery
  W->>A: runSingleTask(ANSWER_QUERY, corpus+question)
  A->>QT: retrieveChunks(question, 5)
  QT->>VS: keyword lookup
  VS-->>QT: List<IndexedChunk>
  QT-->>A: List<IndexedChunk>
  A->>QT: buildContext(chunks)
  QT-->>A: String context
  A->>G: before-agent-response(RagAnswer)
  G->>E: check corpus citations
  G-->>A: accept (all citations grounded)
  G->>E: recordGuardrailOutcome("PASSED")
  A-->>W: RagAnswer
  W->>E: recordAnswer
  E-.->>U: SSE event(ANSWERED)
```

## State machine — `RagSessionEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> INDEXING: IngestStarted
  INDEXING --> INDEXED: CorpusIndexed
  INDEXED --> ANSWERING: QueryStarted
  ANSWERING --> ANSWERED: AnswerDrafted + GuardrailApplied(PASSED)
  ANSWERING --> BLOCKED: AnswerBlocked (budget exhausted)
  INDEXING --> FAILED: SessionFailed
  ANSWERING --> FAILED: SessionFailed
  ANSWERED --> [*]
  BLOCKED --> [*]
  FAILED --> [*]
```

`GuardrailApplied{verdict: BLOCKED}` is recorded each time the guardrail rejects a draft answer during the agent's iteration budget. The status stays `ANSWERING` until the agent either produces a grounded answer (transitions to `ANSWERED`) or exhausts its budget (transitions to `BLOCKED`).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  RagSessionEntity ||--o{ SessionCreated : emits
  RagSessionEntity ||--o{ IngestStarted : emits
  RagSessionEntity ||--o{ CorpusIndexed : emits
  RagSessionEntity ||--o{ QueryStarted : emits
  RagSessionEntity ||--o{ AnswerDrafted : emits
  RagSessionEntity ||--o{ GuardrailApplied : emits
  RagSessionEntity ||--o{ AnswerBlocked : emits
  RagSessionEntity ||--o{ SessionFailed : emits
  RagSessionView }o--|| RagSessionEntity : projects
  RagPipelineWorkflow }o--|| RagSessionEntity : reads-and-writes
  RagAgent ||--o{ IndexedCorpus : returns
  RagAgent ||--o{ RagAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RagEndpoint` | `api/RagEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `RagSessionEntity` | `application/RagSessionEntity.java` (state in `domain/RagSessionRecord.java`, events in `domain/RagSessionEvent.java`) |
| `RagPipelineWorkflow` | `application/RagPipelineWorkflow.java` |
| `RagAgent` | `application/RagAgent.java` (tasks in `application/RagTasks.java`) |
| `IngestTools` | `application/IngestTools.java` |
| `QueryTools` | `application/QueryTools.java` |
| `VectorStore` | `application/VectorStore.java` |
| `AnswerGuardrail` | `application/AnswerGuardrail.java` |
| `RagSessionView` | `application/RagSessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `ingestStep` 60 s, `queryStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(RagPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"rag-" + sessionId` as the workflow id; restart of the same sessionId is rejected by the workflow runtime. The agent instance id is `"agent-" + sessionId` so each session has its own per-task conversation memory.
- **One agent per session**: `RagAgent` runs two tasks per session — INGEST and QUERY — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a draft answer and still let the agent self-correct.
- **Guardrail-driven retry**: when `AnswerGuardrail` rejects a draft answer, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations produce ungrounded citations, the workflow step fails over to `error` and the entity transitions to `BLOCKED` then `FAILED`.
- **VectorStore is in-process and shared across tasks**: the same singleton is written by `IngestTools.indexChunks` in the INGEST task and read by `QueryTools.retrieveChunks` in the QUERY task. Because `ingestStep` writes `CorpusIndexed` and the workflow advances to `queryStep` only after that write, there is no concurrency hazard — the write is complete before the read begins.
- **Task-boundary handoff is the dependency contract**: `ingestStep` writes `CorpusIndexed` BEFORE returning; `queryStep` reads the recorded `IndexedCorpus` from the entity to build the QUERY task's instruction context. The agent itself is stateless across phases — it never holds ingest + query context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed session stays at the last successful event; the UI shows the partial state.
