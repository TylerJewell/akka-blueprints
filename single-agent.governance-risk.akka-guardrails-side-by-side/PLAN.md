# PLAN — guardrails-side-by-side

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
  classDef eval fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[PromptEndpoint]:::ep
  Entity[PromptEntity]:::ese
  WF[GuardrailWorkflow]:::wf
  Screener[InputScreenerAgent]:::agent
  Policy[PolicyAgent]:::agent
  Validator[OutputValidatorAgent]:::agent
  Evaluator[ResponseEvaluator]:::eval
  View[PromptView]:::view
  App[AppEndpoint]:::ep

  API -->|receive| Entity
  API -->|start workflow| WF
  WF -->|inputScreenStep| Screener
  Screener -->|ScreeningVerdict PASS| WF
  Screener -->|ScreeningVerdict BLOCK| WF
  WF -->|BLOCK: recordBlock| Entity
  WF -->|PASS: agentCallStep| Policy
  Policy -->|PolicyResponse| WF
  WF -->|outputValidateStep| Validator
  Validator -->|ValidationVerdict PASS| WF
  Validator -->|ValidationVerdict BLOCK| WF
  WF -->|BLOCK: recordResponseBlock| Entity
  WF -->|PASS: evalStep| Evaluator
  Evaluator -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as PromptEndpoint
  participant E as PromptEntity
  participant W as GuardrailWorkflow
  participant IS as InputScreenerAgent
  participant P as PolicyAgent
  participant OV as OutputValidatorAgent
  participant Ev as ResponseEvaluator

  U->>API: POST /api/prompts
  API->>E: receive(request)
  E-->>API: { promptId }
  API->>W: start(promptId)
  W->>IS: runSingleTask(SCREEN_INPUT, promptText)
  IS-->>W: ScreeningVerdict PASS
  W->>E: startScreening / recordScreenPass
  W->>P: runSingleTask(ANSWER_PROMPT, promptText)
  P-->>W: PolicyResponse
  W->>E: startAgentCall / recordAgentResponse
  W->>OV: runSingleTask(VALIDATE_OUTPUT, reply)
  OV-->>W: ValidationVerdict PASS
  W->>E: startValidation / acceptResponse
  W->>Ev: score(response, request)
  Ev-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(RESPONDED)
```

## State machine — `PromptEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SCREENING: ScreeningStarted
  SCREENING --> BLOCKED: PromptBlocked (input guardrail)
  SCREENING --> AGENT_RUNNING: AgentCallStarted
  AGENT_RUNNING --> VALIDATING: AgentResponded
  VALIDATING --> RESPONSE_BLOCKED: ResponseBlocked (output guardrail)
  VALIDATING --> RESPONDED: ResponseAccepted + EvaluationScored
  AGENT_RUNNING --> FAILED: PromptFailed (agent error)
  RECEIVED --> FAILED: PromptFailed (workflow error)
  BLOCKED --> [*]
  RESPONSE_BLOCKED --> [*]
  RESPONDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PromptEntity ||--o{ PromptReceived : emits
  PromptEntity ||--o{ ScreeningStarted : emits
  PromptEntity ||--o{ PromptBlocked : emits
  PromptEntity ||--o{ AgentCallStarted : emits
  PromptEntity ||--o{ AgentResponded : emits
  PromptEntity ||--o{ ValidationStarted : emits
  PromptEntity ||--o{ ResponseBlocked : emits
  PromptEntity ||--o{ ResponseAccepted : emits
  PromptEntity ||--o{ EvaluationScored : emits
  PromptEntity ||--o{ PromptFailed : emits
  PromptView }o--|| PromptEntity : projects
  GuardrailWorkflow }o--|| PromptEntity : reads-and-writes
  InputScreenerAgent ||--o{ ScreeningVerdict : returns
  PolicyAgent ||--o{ PolicyResponse : returns
  OutputValidatorAgent ||--o{ ValidationVerdict : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PromptEndpoint` | `api/PromptEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PromptEntity` | `application/PromptEntity.java` (state in `domain/Prompt.java`, events in `domain/PromptEvent.java`) |
| `GuardrailWorkflow` | `application/GuardrailWorkflow.java` |
| `InputScreenerAgent` | `application/InputScreenerAgent.java` |
| `PolicyAgent` | `application/PolicyAgent.java` |
| `OutputValidatorAgent` | `application/OutputValidatorAgent.java` |
| `ResponseEvaluator` | `application/ResponseEvaluator.java` |
| `GuardrailTasks` | `application/GuardrailTasks.java` |
| `PromptView` | `application/PromptView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `inputScreenStep` 30 s, `agentCallStep` 60 s, `outputValidateStep` 30 s, `evalStep` 5 s, `blockStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(GuardrailWorkflow::error)` on `agentCallStep`. The 60 s on `agentCallStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"guardrail-" + promptId` as the workflow id; `PromptEndpoint` mints the `promptId` before starting the workflow, so a duplicate POST returns the same `promptId` without starting a second workflow.
- **Guardrail agents are single-iteration**: `InputScreenerAgent` and `OutputValidatorAgent` are configured with `maxIterationsPerTask(1)`. They make one decision and return. They do not retry on their own.
- **Short-circuit on block**: when `inputScreenStep` returns a BLOCK verdict, the workflow transitions immediately to `blockStep` and never calls `PolicyAgent`. The entity history records the screening verdict even on the blocked path.
- **Eval is synchronous and deterministic**: `ResponseEvaluator` runs in-process inside `evalStep`. No LLM call, no external service — the same response always scores the same.
- **No saga / no compensation**: every step is either a single-task agent call, an entity write, or a pure computation. There is nothing external to roll back.
