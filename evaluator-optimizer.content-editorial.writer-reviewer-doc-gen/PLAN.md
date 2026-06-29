# PLAN — writer-reviewer-doc-gen

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

  Writer[WriterAgent]:::agent
  Reviewer[ReviewerAgent]:::agent

  WF[DocumentWorkflow]:::wf
  Doc[DocumentEntity]:::ese
  Queue[RequestQueue]:::ese
  View[DocumentsView]:::view
  Consumer[DocRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[DocumentEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue topic| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|write / revise| Writer
  WF -->|review| Reviewer
  WF -->|emit events| Doc
  Doc -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Doc
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as DocumentEndpoint
  participant Q as RequestQueue
  participant C as DocRequestConsumer
  participant W as DocumentWorkflow
  participant WA as WriterAgent
  participant R as ReviewerAgent
  participant E as DocumentEntity
  participant V as DocumentsView

  U->>API: POST /api/documents {topic, wordCeiling}
  API->>Q: append TopicSubmitted
  API-->>U: 202 {documentId}
  Q->>C: TopicSubmitted
  C->>W: start({documentId, topic, wordCeiling, maxAttempts=4})
  W->>E: emit DocumentCreated (DRAFTING)

  W->>WA: WRITE(topic, wordCeiling)
  WA-->>W: DocumentDraft #1 (480 words)
  W->>E: emit DraftProduced (n=1)
  Note over W: guardrailStep (deterministic word-count check)
  W->>E: emit DraftGuardrailVerdictRecorded (passed=true)
  W->>E: status REVIEWING
  W->>R: REVIEW(DocumentDraft #1)
  R-->>W: Review{REVISE, score=3, 3 bullets}
  W->>E: emit DraftReviewed (n=1, REVISE)

  W->>WA: REVISE_DRAFT(topic, prior, notes)
  WA-->>W: DocumentDraft #2 (450 words)
  W->>E: emit DraftProduced (n=2)
  W->>E: emit DraftGuardrailVerdictRecorded (passed=true)
  W->>R: REVIEW(DocumentDraft #2)
  R-->>W: Review{APPROVE, score=5, rationale}
  W->>E: emit DraftReviewed (n=2, APPROVE)
  W->>E: emit DocumentApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `DocumentEntity`

```mermaid
stateDiagram-v2
  [*] --> DRAFTING
  DRAFTING --> REVIEWING: DraftProduced + guardrail passed
  DRAFTING --> DRAFTING: guardrail blocked, re-draft
  REVIEWING --> DRAFTING: Review = REVISE, attempts < max
  REVIEWING --> APPROVED: Review = APPROVE
  REVIEWING --> REJECTED_FINAL: Review = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  DocumentEntity ||--o{ DocumentCreated : emits
  DocumentEntity ||--o{ DraftProduced : emits
  DocumentEntity ||--o{ DraftGuardrailVerdictRecorded : emits
  DocumentEntity ||--o{ DraftReviewed : emits
  DocumentEntity ||--o{ DocumentApproved : emits
  DocumentEntity ||--o{ DocumentRejectedFinal : emits
  DocumentEntity ||--o{ EvalRecorded : emits
  DocumentsView }o--|| DocumentEntity : projects
  RequestQueue ||--o{ TopicSubmitted : emits
  DocRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `WriterAgent` | `application/WriterAgent.java` |
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `DocTasks` | `application/DocTasks.java` |
| `DocumentWorkflow` | `application/DocumentWorkflow.java` |
| `DocumentEntity` | `application/DocumentEntity.java` (state in `domain/Document.java`, events in `domain/DocumentEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `DocumentsView` | `application/DocumentsView.java` |
| `DocRequestConsumer` | `application/DocRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `DocumentEndpoint` | `api/DocumentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `writeStep` and `reviewStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `DocumentEndpoint.submit` uses `(topic, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(documentId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `writer-reviewer-doc-gen.refinement.max-attempts` (default 4). The workflow checks the count BEFORE calling `writeStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt at `REJECTED_FINAL` is the only terminal fallback; it preserves the best draft and every review on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it counts words in the draft and either advances to `reviewStep` or returns to `writeStep` with a structured feedback note. The structured feedback never becomes an LLM-generated review; it stays a deterministic `ReviewNotes` payload with a single bullet.
