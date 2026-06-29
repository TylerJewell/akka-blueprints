# PLAN — personal-finance-agent

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
  classDef tools fill:#0e2020,stroke:#06b6d4,color:#06b6d4;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  Sanitizer[TransactionSanitizer]:::cons
  WF[QueryWorkflow]:::wf
  Agent[FinanceAssistantAgent]:::agent
  Guard[WriteGuardrail]:::guard
  Tools[FinanceTools]:::tools
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|QuerySubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -->|tool call| Tools
  Agent -.->|before-tool-call write| Guard
  Guard -->|approve / block| Agent
  Agent -->|AssistantResponse| WF
  WF -->|recordAnswer| Entity
  WF -->|finish| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (spending summary, happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant S as TransactionSanitizer
  participant W as QueryWorkflow
  participant A as FinanceAssistantAgent
  participant G as WriteGuardrail
  participant T as FinanceTools

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  E-.->>S: QuerySubmitted
  S->>S: tokenise PII in transactions
  S->>E: attachSanitized
  S->>W: start(queryId)
  W->>E: poll getQuery
  E-->>W: sanitized.isPresent()
  W->>E: markAnswering
  W->>A: runSingleTask(query + context)
  A->>T: listTransactions(accountId, ...)
  T-->>A: List<SanitizedTransaction>
  A->>T: groupByCategory(transactions)
  T-->>A: List<CategoryTotal>
  A-->>W: AssistantResponse
  W->>E: recordAnswer(response)
  W->>E: finish()
  E-.->>U: SSE event(ANSWERED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: TransactionsSanitized
  SANITIZED --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWERED: AnswerRecorded
  SUBMITTED --> FAILED: QueryFailed (sanitizer error)
  ANSWERING --> FAILED: QueryFailed (agent error)
  ANSWERED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ TransactionsSanitized : emits
  QueryEntity ||--o{ AnsweringStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  TransactionSanitizer }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  FinanceAssistantAgent ||--o{ AssistantResponse : returns
  FinanceTools ||--o{ ToolCallRecord : produces
  WriteGuardrail ||--o{ WriteOutcome : decides
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `TransactionSanitizer` | `application/TransactionSanitizer.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `FinanceAssistantAgent` | `application/FinanceAssistantAgent.java` (tasks in `application/FinanceTasks.java`) |
| `FinanceTools` | `application/FinanceTools.java` |
| `WriteGuardrail` | `application/WriteGuardrail.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `answerStep` 90 s (accommodates multiple tool-call round trips plus LLM latency — Lesson 4), `recordStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`.
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; `TransactionSanitizer` is allowed to redeliver `QuerySubmitted` events because `QueryEntity.attachSanitized` is event-version-guarded — a second sanitize call against an already-sanitized query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"agent-" + queryId`, which gives each task its own conversation context. `capability(...).maxIterationsPerTask(5)` allows multiple tool-call rounds per query (typical finance queries call 2–3 tools).
- **Guardrail on write only**: `WriteGuardrail` inspects only `transferFunds` and `payBill` invocations. Read tools bypass it entirely. This is encoded in the tool registration metadata so the guardrail hook does not need a runtime name-check inside its body.
- **No compensation**: `getBalance` and `listTransactions` are read-only; `transferFunds` and `payBill` mutate the in-memory simulation only. There is no real bank connection and no saga rollback needed. A `FAILED` query's prior tool-call trace is preserved on the entity for audit.
- **Eval**: this baseline has no on-decision evaluator (only two controls: sanitizer + guardrail). The single-agent invariant is upheld — no second LLM call anywhere in the stack.
