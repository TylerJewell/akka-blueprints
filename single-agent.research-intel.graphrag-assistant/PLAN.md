# PLAN — graphrag-assistant

Architectural sketch. All four mermaid diagrams + the component table.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef kve fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  QE[QueryEndpoint]:::ep
  AE[AppEndpoint]:::ep
  RA[ResearchAgent]:::agent
  CI[CorpusIndex]:::kve
  QENT[QueryEntity]:::ese
  QV[QueriesView]:::view
  IB[IndexBuilder]:::ta

  QE -->|answer| RA
  RA -->|localSearch / globalSearch| CI
  QE -->|receive / recordAnswer / block| QENT
  QENT -.->|events| QV
  QE -->|getAllQueries / SSE| QV
  IB -.->|tick: build| CI
  AE -->|static| AE
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant QE as QueryEndpoint
  participant QN as QueryEntity
  participant RA as ResearchAgent
  participant CI as CorpusIndex
  U->>QE: POST /api/ask {question}
  QE->>QN: receive(question)
  QN-->>QE: queryId (RECEIVED)
  QE->>RA: answer(question)
  RA->>CI: localSearch or globalSearch(query)
  CI-->>RA: RetrievalResult {chunks, citations}
  Note over RA: sanitizer S1 scrubs PII from chunks before prompt
  RA-->>QE: Answer {text, scope, grounded, citations}
  Note over QE: guardrail G1 checks grounding
  alt grounded
    QE->>QN: recordAnswer(Answer)
    QN-->>QE: ANSWERED
  else not grounded
    QE->>QN: block(reason)
    QN-->>QE: BLOCKED
  end
  QE-->>U: SSE update
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> ANSWERED: recordAnswer (grounded)
  RECEIVED --> BLOCKED: block (ungrounded)
  ANSWERED --> [*]
  BLOCKED --> [*]
```

## Entity model

```mermaid
erDiagram
  QUERY_ENTITY ||--o{ QUERY_EVENT : emits
  QUERY_ENTITY ||--|| QUERIES_VIEW : projects
  CORPUS_INDEX ||--o{ DOC : indexes
  QUERY_ENTITY {
    string id
    string question
    enum status
    string scope
    string answer
    bool grounded
  }
  QUERY_EVENT {
    string type
  }
  QUERIES_VIEW {
    string id
    enum status
  }
  CORPUS_INDEX {
    bool built
    int docCount
    int entityCount
    int communityCount
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| ResearchAgent | `application/ResearchAgent.java` |
| CorpusIndex | `application/CorpusIndex.java` |
| QueryEntity | `application/QueryEntity.java` |
| QueriesView | `application/QueriesView.java` |
| IndexBuilder | `application/IndexBuilder.java` |
| QueryEndpoint | `api/QueryEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Query / Answer / RetrievalResult / IndexState | `domain/*.java` |

## Concurrency notes

- The `QueryEndpoint.ask` handler calls `ResearchAgent.answer`, which makes an
  LLM call; set the component-client invoke timeout to at least 60 seconds so the
  call does not hit the 5-second default (Lesson 4).
- `queryId` is the idempotency key: `ask` generates a fresh UUID; re-posting the
  same question creates a distinct query, so duplicate submissions are isolated.
- No saga / compensation: the flow is request/response with a single terminal
  transition (`ANSWERED` or `BLOCKED`). The guardrail decides which.
- `IndexBuilder` is idempotent: it checks `CorpusIndex.getStatus().built` before
  rebuilding, so repeated ticks do not re-index.
