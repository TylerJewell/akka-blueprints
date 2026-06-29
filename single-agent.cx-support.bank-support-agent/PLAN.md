# PLAN — bank-support-agent

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
  classDef tool fill:#0e1a1a,stroke:#26c6da,color:#26c6da;

  API[EnquiryEndpoint]:::ep
  Entity[EnquiryEntity]:::ese
  WF[EnquiryWorkflow]:::wf
  Agent[BankSupportAgent]:::agent
  ToolGuard[ToolCallGuardrail]:::guard
  RespGuard[ResponseGuardrail]:::guard
  Tool[AccountLookupTool]:::tool
  Sanitizer[ResponseSanitizer]:::cons
  View[EnquiryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|lookupStep| Tool
  Tool -->|AccountSummary| WF
  WF -->|loadAccount| Entity
  WF -->|respondStep runSingleTask| Agent
  Agent -.->|before-tool-call| ToolGuard
  ToolGuard -.->|allow / block| Agent
  Agent -->|AccountLookupTool call| Tool
  Agent -.->|before-agent-response| RespGuard
  RespGuard -.->|accept / reject| Agent
  Agent -->|SupportResponse| WF
  WF -->|recordResponse| Entity
  WF -->|sanitizeStep| Sanitizer
  Sanitizer -->|attachSanitizedLog| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as EnquiryEndpoint
  participant E as EnquiryEntity
  participant WF as EnquiryWorkflow
  participant T as AccountLookupTool
  participant A as BankSupportAgent
  participant TG as ToolCallGuardrail
  participant RG as ResponseGuardrail
  participant S as ResponseSanitizer

  U->>API: POST /api/enquiries
  API->>E: submit(enquiry)
  API->>WF: start(enquiryId)
  E-->>API: { enquiryId }
  WF->>T: lookup(customerId)
  T-->>WF: AccountSummary
  WF->>E: loadAccount(account)
  WF->>E: startResponding
  WF->>A: runSingleTask(context)
  A->>TG: before-tool-call(lookup params)
  TG-->>A: allow
  A->>T: AccountLookupTool.lookup(customerId)
  T-->>A: AccountSummary
  A->>RG: before-agent-response(candidate)
  RG-->>A: accept
  A-->>WF: SupportResponse
  WF->>E: recordResponse(response)
  WF->>S: sanitize(response.answer)
  S->>E: attachSanitizedLog(sanitizedLog)
  E-.->>U: SSE event(LOGGED)
```

## State machine — `EnquiryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> ACCOUNT_LOADED: AccountLoaded
  ACCOUNT_LOADED --> RESPONDING: RespondingStarted
  RESPONDING --> RESPONSE_RECORDED: ResponseRecorded
  RESPONSE_RECORDED --> LOGGED: LogSanitized
  ACCOUNT_LOADED --> FAILED: EnquiryFailed (lookup error)
  RESPONDING --> FAILED: EnquiryFailed (agent error)
  LOGGED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  EnquiryEntity ||--o{ EnquirySubmitted : emits
  EnquiryEntity ||--o{ AccountLoaded : emits
  EnquiryEntity ||--o{ RespondingStarted : emits
  EnquiryEntity ||--o{ ResponseRecorded : emits
  EnquiryEntity ||--o{ LogSanitized : emits
  EnquiryEntity ||--o{ EnquiryFailed : emits
  EnquiryView }o--|| EnquiryEntity : projects
  ResponseSanitizer }o--|| EnquiryEntity : subscribes
  EnquiryWorkflow }o--|| EnquiryEntity : reads-and-writes
  BankSupportAgent ||--o{ SupportResponse : returns
  AccountLookupTool ||--o{ AccountSummary : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `EnquiryEndpoint` | `api/EnquiryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `EnquiryEntity` | `application/EnquiryEntity.java` (state in `domain/Enquiry.java`, events in `domain/EnquiryEvent.java`) |
| `EnquiryWorkflow` | `application/EnquiryWorkflow.java` |
| `BankSupportAgent` | `application/BankSupportAgent.java` (tasks in `application/EnquiryTasks.java`) |
| `AccountLookupTool` | `application/AccountLookupTool.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `ResponseSanitizer` | `application/ResponseSanitizer.java` |
| `EnquiryView` | `application/EnquiryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `lookupStep` 10 s, `respondStep` 60 s, `sanitizeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(EnquiryWorkflow::error)`. The 60 s on `respondStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"enquiry-" + enquiryId` as the workflow id; the `EnquiryEndpoint` also passes `enquiryId` to `EnquiryEntity.submit`, which is event-version-guarded — a duplicate submit is a no-op.
- **One agent per enquiry**: the AutonomousAgent instance id is `"support-" + enquiryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ResponseGuardrail` rejects a candidate response, the rejection returns as a structured error to the agent loop. If all 3 iterations fail validation, the workflow's `respondStep` fails over to `error` and the entity transitions to `FAILED`.
- **Tool-call veto**: when `ToolCallGuardrail` blocks a `blockCard=true` call, the block is returned to the agent as a pre-condition error. The agent must re-plan — it cannot call `blockCardRequest` directly.
- **Sanitizer runs synchronously in sanitizeStep**: unlike the docreview pattern where a Consumer fires asynchronously after entity events, here the sanitizer is invoked directly inside `sanitizeStep`. This keeps the lifecycle tightly ordered: `RESPONSE_RECORDED → LOGGED` happens in a single workflow step, preventing a window where the raw answer is visible in the view.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
