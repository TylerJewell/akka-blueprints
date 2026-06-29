# PLAN — rag-pdf-chat

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ChatEndpoint]:::ep
  Entity[PdfDocumentEntity]:::ese
  Retriever[PassageRetriever]:::cons
  WF[ChatSessionWorkflow]:::wf
  Agent[PdfChatAgent]:::agent
  Guard[CitationGuardrail]:::guard
  View[ChatSessionView]:::view
  App[AppEndpoint]:::ep

  API -->|upload| Entity
  Entity -.->|DocumentUploaded| Retriever
  Retriever -->|attachIndex| Entity
  API -->|ask question| WF
  WF -->|retrieveStep rankPassages| Retriever
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|CitedAnswer| WF
  WF -->|AnswerRecorded| View
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ChatEndpoint
  participant E as PdfDocumentEntity
  participant R as PassageRetriever
  participant W as ChatSessionWorkflow
  participant A as PdfChatAgent
  participant G as CitationGuardrail

  U->>API: POST /api/documents (PDF upload)
  API->>E: upload(filename, pdfBase64)
  E-->>API: { documentId }
  E-.->>R: DocumentUploaded
  R->>R: chunk + index passages
  R->>E: attachIndex(metadata, passages)
  E-->>U: SSE INDEXED

  U->>API: POST /api/documents/{id}/chat (question)
  API->>W: start(questionId, documentId, questionText)
  W->>R: rankPassages(documentId, questionText, 5)
  R-->>W: top-5 PdfPassage list
  W->>A: runSingleTask(question + passages attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: CitedAnswer
  W->>View: AnswerRecorded
  View-->>U: SSE ANSWERED
```

## State machine — `PdfDocumentEntity`

```mermaid
stateDiagram-v2
  [*] --> UPLOADING
  UPLOADING --> INDEXED: DocumentIndexed
  UPLOADING --> FAILED: DocumentIndexFailed
  INDEXED --> [*]
  FAILED --> [*]
```

## Exchange state machine — `ChatSessionWorkflow`

```mermaid
stateDiagram-v2
  [*] --> RETRIEVING
  RETRIEVING --> ANSWERING: PassagesRetrieved
  ANSWERING --> ANSWERED: AnswerRecorded (answerable=true)
  ANSWERING --> UNANSWERABLE: AnswerRecorded (answerable=false)
  RETRIEVING --> FAILED: ExchangeFailed (retriever error)
  ANSWERING --> FAILED: ExchangeFailed (agent error / guardrail-exhaustion)
  ANSWERED --> [*]
  UNANSWERABLE --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PdfDocumentEntity ||--o{ DocumentUploaded : emits
  PdfDocumentEntity ||--o{ DocumentIndexed : emits
  PdfDocumentEntity ||--o{ DocumentIndexFailed : emits
  ChatSessionView }o--|| PdfDocumentEntity : projects
  PassageRetriever }o--|| PdfDocumentEntity : subscribes
  ChatSessionWorkflow }o--|| PassageRetriever : calls-rankPassages
  PdfChatAgent ||--o{ CitedAnswer : returns
  ChatSessionWorkflow }o--|| PdfDocumentEntity : reads
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PdfDocumentEntity` | `application/PdfDocumentEntity.java` (state in `domain/PdfDocument.java`, events in `domain/PdfDocumentEvent.java`) |
| `PassageRetriever` | `application/PassageRetriever.java` |
| `ChatSessionWorkflow` | `application/ChatSessionWorkflow.java` |
| `PdfChatAgent` | `application/PdfChatAgent.java` (tasks in `application/PdfChatTasks.java`) |
| `CitationGuardrail` | `application/CitationGuardrail.java` |
| `ChatSessionView` | `application/ChatSessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `retrieveStep` 10 s, `answerStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ChatSessionWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"chat-" + questionId` as the workflow id; the `PassageRetriever` Consumer is allowed to redeliver `DocumentUploaded` events because `PdfDocumentEntity.attachIndex` is event-version-guarded — a second indexing attempt against an already-indexed document is a no-op.
- **One agent per question**: the AutonomousAgent instance id is `"chat-" + questionId`, which gives each question its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `CitationGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `answerStep` fails over to `error` and the exchange transitions to `FAILED`.
- **Retriever is synchronous and deterministic**: `PassageRetriever.rankPassages` runs in-process. No LLM call, no external service. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
