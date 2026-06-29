# PLAN — guardrails-pattern

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[InteractionEndpoint]:::ep
  Entity[InteractionEntity]:::ese
  WF[InteractionWorkflow]:::wf
  Agent[AssistantAgent]:::agent
  PG[PromptGuardrail]:::guard
  RG[ReplyGuardrail]:::guard
  View[InteractionView]:::view
  App[AppEndpoint]:::ep

  API -->|submit + start workflow| Entity
  API -->|start| WF
  WF -->|promptGuardStep check| PG
  PG -->|PASS / BLOCK| WF
  WF -->|recordPromptGuardResult| Entity
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-llm-call| PG
  Agent -.->|before-agent-response| RG
  Agent -->|AgentReply| WF
  WF -->|recordReply| Entity
  WF -->|replyGuardStep check| RG
  RG -->|PASS / REJECT| WF
  WF -->|recordReplyGuardResult| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as InteractionEndpoint
  participant E as InteractionEntity
  participant W as InteractionWorkflow
  participant PG as PromptGuardrail
  participant A as AssistantAgent
  participant RG as ReplyGuardrail

  U->>API: POST /api/interactions
  API->>E: submit(request)
  API->>W: start(interactionId)
  E-->>API: { interactionId }
  W->>PG: check(promptText, policyProfile)
  PG-->>W: PASS
  W->>E: recordPromptGuardResult(passed=true)
  W->>A: runSingleTask(prompt + profile)
  A->>PG: before-llm-call(prompt)
  PG-->>A: allow
  A->>RG: before-agent-response(candidate)
  RG-->>A: accept
  A-->>W: AgentReply
  W->>E: recordReply(reply)
  W->>RG: check(reply, policyProfile)
  RG-->>W: PASS
  W->>E: recordReplyGuardResult(passed=true)
  E-.->>U: SSE event(REPLY_RECORDED)
```

## State machine — `InteractionEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PROMPT_CHECKED: PromptGuardChecked (passed)
  SUBMITTED --> BLOCKED: PromptBlocked
  PROMPT_CHECKED --> REPLYING: ReplyingStarted
  REPLYING --> REPLY_RECORDED: ReplyRecorded
  REPLYING --> FAILED: InteractionFailed (agent / guard exhaustion)
  BLOCKED --> [*]
  REPLY_RECORDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  InteractionEntity ||--o{ PromptSubmitted : emits
  InteractionEntity ||--o{ PromptGuardChecked : emits
  InteractionEntity ||--o{ PromptBlocked : emits
  InteractionEntity ||--o{ ReplyingStarted : emits
  InteractionEntity ||--o{ ReplyRecorded : emits
  InteractionEntity ||--o{ ReplyGuardChecked : emits
  InteractionEntity ||--o{ InteractionFailed : emits
  InteractionView }o--|| InteractionEntity : projects
  InteractionWorkflow }o--|| InteractionEntity : reads-and-writes
  AssistantAgent ||--o{ AgentReply : returns
  PromptGuardrail ||--o{ GuardrailOutcome : produces
  ReplyGuardrail ||--o{ GuardrailOutcome : produces
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `InteractionEndpoint` | `api/InteractionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `InteractionEntity` | `application/InteractionEntity.java` (state in `domain/Interaction.java`, events in `domain/InteractionEvent.java`) |
| `InteractionWorkflow` | `application/InteractionWorkflow.java` |
| `AssistantAgent` | `application/AssistantAgent.java` (tasks in `application/AssistantTasks.java`) |
| `PromptGuardrail` | `application/PromptGuardrail.java` |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `InteractionView` | `application/InteractionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `promptGuardStep` 5 s, `agentStep` 60 s, `replyGuardStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(InteractionWorkflow::error)`. The 60 s on `agentStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"interaction-" + interactionId` as the workflow id. The `InteractionEntity.submit` command is version-guarded — a duplicate submission is a no-op.
- **One agent per interaction**: the AutonomousAgent instance id is `"assistant-" + interactionId`, giving each task its own conversation context. `maxIterationsPerTask(3)` caps guardrail-triggered retries.
- **Two guardrail hooks, one agent**: `PromptGuardrail` is bound to `before-llm-call`; `ReplyGuardrail` is bound to `before-agent-response`. Both are registered on `AssistantAgent`. The workflow's `replyGuardStep` calls `ReplyGuardrail.check()` a second time for record-keeping — it does not invoke a second agent.
- **Blocked interactions are terminal**: when `PromptGuardrail` blocks, the entity transitions to `BLOCKED` and no model call is made. This is the key property demonstrated: the model never sees a non-compliant prompt.
- **No saga / no compensation**: every step is either a pure check, a direct entity write, or a single-task agent call. There is nothing external to roll back.
