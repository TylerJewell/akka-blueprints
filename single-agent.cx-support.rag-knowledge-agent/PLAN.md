# PLAN — rag-knowledge-agent

Architectural sketch. All four mermaid diagrams + the component table.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef kve fill:#13202a,stroke:#38bdf8,color:#38bdf8;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Browser --> KnowledgeEndpoint
  Browser --> AppEndpoint
  AppEndpoint --> Static[static-resources]
  KnowledgeEndpoint -->|search| DocIndexEntity
  KnowledgeEndpoint -->|answer| KnowledgeAgent
  KnowledgeEndpoint -->|record| QuerySessionEntity
  KnowledgeEndpoint -->|getAllSessions| QuerySessionView
  DocLoader -.->|indexChunks tick| DocIndexEntity
  QuerySessionEntity -.->|events| QuerySessionView
  QuerySessionEntity -.->|events| FaithfulnessEvalConsumer
  FaithfulnessEvalConsumer -->|score| FaithfulnessAgent
  FaithfulnessEvalConsumer -->|recordEvaluation| QuerySessionEntity

  class KnowledgeAgent,FaithfulnessAgent agent;
  class DocIndexEntity kve;
  class QuerySessionEntity ese;
  class QuerySessionView view;
  class FaithfulnessEvalConsumer cons;
  class DocLoader ta;
  class KnowledgeEndpoint,AppEndpoint ep;
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant E as KnowledgeEndpoint
  participant Q as QuerySessionEntity
  participant D as DocIndexEntity
  participant K as KnowledgeAgent
  participant C as FaithfulnessEvalConsumer
  participant F as FaithfulnessAgent
  U->>E: POST /api/ask {question}
  E->>Q: receive(question)  [RECEIVED]
  E->>D: search(embed(question), topK)
  D-->>E: top citations
  E->>Q: recordRetrieval(chunks)  [RETRIEVED]
  E->>K: answer(question, chunks)
  Note over K: grounding guardrail (before-agent-response)
  alt grounded
    K-->>E: GroundedAnswer
    E->>Q: recordAnswer(answer, citations)  [ANSWERED]
    Q-->>C: QueryAnswered
    C->>F: score(answer, chunks)
    F-->>C: FaithfulnessResult
    C->>Q: recordEvaluation(score, verdict)  [EVALUATED]
  else unsupported
    K-->>E: refusal
    E->>Q: recordRefusal(reason)  [REFUSED]
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> RETRIEVED: ChunksRetrieved
  RETRIEVED --> ANSWERED: QueryAnswered
  RETRIEVED --> REFUSED: QueryRefused
  ANSWERED --> EVALUATED: FaithfulnessEvaluated
  REFUSED --> [*]
  EVALUATED --> [*]
```

## Entity model

```mermaid
erDiagram
  QUERY_SESSION ||--o{ QUERY_SESSION_EVENT : emits
  QUERY_SESSION ||--o{ CITATION : cites
  DOC_INDEX ||--o{ CHUNK : holds
  QUERY_SESSION_EVENT {
    string type "QueryReceived|ChunksRetrieved|QueryAnswered|QueryRefused|FaithfulnessEvaluated"
  }
  QUERY_SESSION {
    string id
    string question
    string status
    double faithfulnessScore
  }
  CHUNK {
    string docId
    string chunkId
    string text
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| KnowledgeAgent | `application/KnowledgeAgent.java` |
| FaithfulnessAgent | `application/FaithfulnessAgent.java` |
| DocIndexEntity | `application/DocIndexEntity.java` |
| DocLoader | `application/DocLoader.java` |
| QuerySessionEntity | `application/QuerySessionEntity.java` |
| QuerySessionView | `application/QuerySessionView.java` |
| FaithfulnessEvalConsumer | `application/FaithfulnessEvalConsumer.java` |
| KnowledgeEndpoint | `api/KnowledgeEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Domain records | `domain/*.java` |

## Concurrency notes

- The endpoint's `KnowledgeAgent.answer` call and the consumer's `FaithfulnessAgent.score` call each get a 60s timeout — LLM calls exceed the 5s default (Lesson 4).
- `DocLoader` is idempotent: it no-ops when the index already covers the current files, so repeated ticks do not duplicate chunks.
- `POST /api/ask` uses the new session id as the idempotency key; a retried request with the same id is a no-op on the entity.
- No saga: a refusal is a terminal branch, not a compensation. The faithfulness eval is a downstream non-blocking projection driven by the `QueryAnswered` event, so a slow eval never blocks the answer returned to the user.
