# PLAN — financial-model-builder

---

## 1. Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
    classDef ep    fill:#1a3a4a,stroke:#7EC8E3,color:#fff
    classDef wf    fill:#1a2e1a,stroke:#6dbf6d,color:#fff
    classDef agent fill:#2e1a2e,stroke:#c084fc,color:#fff
    classDef ese   fill:#2e2a1a,stroke:#fbbf24,color:#fff
    classDef view  fill:#1a2a3a,stroke:#60a5fa,color:#fff
    classDef tool  fill:#1a1a2e,stroke:#818cf8,color:#fff
    classDef guard fill:#2e1a1a,stroke:#f87171,color:#fff
    classDef score fill:#1a2e2e,stroke:#34d399,color:#fff

    ME[ModelEndpoint]:::ep
    AE[AppEndpoint]:::ep
    WF[FinancialModelPipelineWorkflow]:::wf
    AG[FinancialModelAgent]:::agent
    ESE[FinancialModelEntity]:::ese
    VIEW[FinancialModelView]:::view
    ET[ExtractTools]:::tool
    BT[BuildTools]:::tool
    VT[ValidateTools]:::tool
    GRD[ModelPhaseGuardrail]:::guard
    SCR[FilingFidelityScorer]:::score

    ME -->|commands| ESE
    ME -->|start pipeline| WF
    WF -->|extractStep| AG
    WF -->|buildStep| AG
    WF -->|validateStep| AG
    WF -->|reviewStep — pause| ESE
    WF -->|evalStep| SCR
    AG --> GRD
    AG --> ET
    AG --> BT
    AG --> VT
    ESE -->|events| VIEW
    AE -->|SSE| VIEW
```

---

## 2. Interaction sequence — J1 happy path

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
    actor User
    participant ME as ModelEndpoint
    participant ESE as FinancialModelEntity
    participant WF as FinancialModelPipelineWorkflow
    participant GRD as ModelPhaseGuardrail
    participant AG as FinancialModelAgent
    participant ET as ExtractTools
    participant BT as BuildTools
    participant VT as ValidateTools
    participant SCR as FilingFidelityScorer

    User->>ME: POST /api/models {ticker, period}
    ME->>ESE: CreateModel command
    ESE-->>WF: ModelCreated → start pipeline
    WF->>AG: extractStep (EXTRACT_FINANCIALS)
    AG->>GRD: before-tool-call check (phase=EXTRACT)
    GRD-->>AG: allowed
    AG->>ET: parseFiling(ticker, period)
    ET-->>AG: List<LineItem>
    AG-->>WF: FilingData
    WF->>ESE: FilingExtracted command
    ESE-->>WF: status = EXTRACTED
    WF->>AG: buildStep (BUILD_MODEL)
    AG->>GRD: before-tool-call check (phase=BUILD)
    GRD-->>AG: allowed
    AG->>BT: computeRatio / projectFigure
    BT-->>AG: List<ModelRow>
    AG-->>WF: FinancialModel
    WF->>ESE: ModelBuilt command
    ESE-->>WF: status = BUILT
    WF->>AG: validateStep (VALIDATE_MODEL)
    AG->>GRD: before-tool-call check (phase=VALIDATE)
    GRD-->>AG: allowed
    AG->>VT: crossCheckAssumption / detectAnomalies
    VT-->>AG: List<ValidationFlag>
    AG-->>WF: ValidationReport
    WF->>ESE: ModelValidated command
    ESE-->>WF: status = PENDING_REVIEW
    Note over WF: reviewStep — pause, wait on entity
    User->>ME: POST /api/models/{id}/approve
    ME->>ESE: ApproveModel command
    ESE-->>WF: ModelApproved → status = APPROVED
    WF->>SCR: evalStep
    SCR-->>ESE: EvaluationScored (score 1-5)
    ESE-->>User: status = EVALUATED
```

---

## 3. State machine — FinancialModelEntity

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
    [*] --> CREATED : ModelCreated
    CREATED --> EXTRACTING : ExtractStarted
    EXTRACTING --> EXTRACTED : FilingExtracted
    EXTRACTED --> BUILDING : BuildStarted
    BUILDING --> BUILT : ModelBuilt
    BUILT --> VALIDATING : ValidateStarted
    VALIDATING --> VALIDATED : ModelValidated
    VALIDATED --> PENDING_REVIEW : ReviewRequested
    PENDING_REVIEW --> APPROVED : ModelApproved
    PENDING_REVIEW --> REJECTED : ModelRejected
    APPROVED --> EVALUATED : EvaluationScored
    EXTRACTING --> FAILED : ModelFailed
    BUILDING --> FAILED : ModelFailed
    VALIDATING --> FAILED : ModelFailed
    REJECTED --> [*]
    EVALUATED --> [*]
    FAILED --> [*]

    note right of PENDING_REVIEW
        GuardrailRejected is audit-only
        — no state transition
    end note
```

---

## 4. Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
erDiagram
    FinancialModelEntity ||--o{ ModelEvent : "emits"
    FinancialModelEntity {
        string modelId PK
        string ticker
        ModelStatus status
        Instant createdAt
    }
    ModelEvent {
        string eventType
        string modelId FK
        Instant occurredAt
    }
    FinancialModelView ||--o{ FinancialModelEntity : "projects from"
    FinancialModelView {
        string modelId PK
        ModelStatus status
        string ticker
        Instant updatedAt
    }
    FinancialModelPipelineWorkflow ||--o{ FinancialModelEntity : "reads and writes"
    FinancialModelPipelineWorkflow {
        string workflowId PK
        string modelId FK
        string currentStep
    }
    FinancialModelAgent ||--|| FilingData : "returns (EXTRACT)"
    FinancialModelAgent ||--|| FinancialModel : "returns (BUILD)"
    FinancialModelAgent ||--|| ValidationReport : "returns (VALIDATE)"
```

---

## 5. Component table

| Java file | Kind | Responsibility |
|---|---|---|
| `entity/FinancialModelEntity.java` | EventSourcedEntity | All state; processes commands; emits events |
| `workflow/FinancialModelPipelineWorkflow.java` | Workflow | Drives extract→build→validate→review→eval steps |
| `agent/FinancialModelAgent.java` | AutonomousAgent | Executes one task per invocation via tools |
| `tools/ExtractTools.java` | Tool class | parseFiling, fetchLineItem |
| `tools/BuildTools.java` | Tool class | computeRatio, projectFigure |
| `tools/ValidateTools.java` | Tool class | crossCheckAssumption, detectAnomalies |
| `guardrail/ModelPhaseGuardrail.java` | Guardrail | before-tool-call phase gate |
| `scorer/FilingFidelityScorer.java` | Scorer | on-decision-eval; four filing-fidelity checks |
| `view/FinancialModelView.java` | View | SSE-ready read-model projected from entity events |
| `endpoint/ModelEndpoint.java` | HttpEndpoint | /api/models/* CRUD + approve/reject + SSE |
| `endpoint/AppEndpoint.java` | HttpEndpoint | /app/* static assets + / redirect |
| `model/ModelTasks.java` | Constants | EXTRACT_FINANCIALS, BUILD_MODEL, VALIDATE_MODEL |

---

## 6. Concurrency notes

- One `FinancialModelEntity` instance per `modelId`; concurrent submissions for different tickers are independent.
- The workflow's `reviewStep` uses `asyncEffect` to observe entity state; it does not hold a thread.
- `FilingFidelityScorer` is synchronous and stateless; it reads from the workflow context, not from a separate entity call.
- `ModelPhaseGuardrail` is read-only; it does not mutate entity state and never emits events directly — only the entity emits `GuardrailRejected` when the workflow reports the block.
