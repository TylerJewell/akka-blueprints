# PLAN — code-assistant

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

  API[EditEndpoint]:::ep
  Entity[EditEntity]:::ese
  Gate[CIGate]:::cons
  WF[EditWorkflow]:::wf
  Agent[CodeAssistantAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  View[EditView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|analyzeStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|block or allow| Agent
  Agent -->|EditPlan| WF
  WF -->|recordPlan| Entity
  Entity -.->|EditProposed| Gate
  Gate -->|recordGateResult| Entity
  WF -->|awaitGateStep poll| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as EditEndpoint
  participant E as EditEntity
  participant W as EditWorkflow
  participant A as CodeAssistantAgent
  participant G as ToolCallGuardrail
  participant CI as CIGate

  U->>API: POST /api/edits
  API->>E: submit(request)
  API->>W: start(editId)
  E-->>API: { editId }
  W->>E: markAnalyzing
  W->>A: runSingleTask(taskDescription + file attachments)
  A->>G: before-tool-call(read_file)
  G-->>A: allow
  A->>G: before-tool-call(run_tests)
  G-->>A: allow
  A-->>W: EditPlan
  W->>E: recordPlan(plan)
  E-.->>CI: EditProposed
  CI->>CI: apply changes + run testCommand
  CI->>E: recordGateResult(gateResult)
  W->>E: poll getEdit
  E-->>W: gateResult.isPresent()
  E-.->>U: SSE event(GATE_PASSED)
```

## State machine — `EditEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> ANALYZING: AnalysisStarted
  ANALYZING --> EDIT_PROPOSED: EditProposed
  EDIT_PROPOSED --> GATE_PASSED: GatePassed
  EDIT_PROPOSED --> GATE_FAILED: GateFailed
  ANALYZING --> FAILED: EditFailed (agent error)
  SUBMITTED --> FAILED: EditFailed (submit error)
  GATE_PASSED --> [*]
  GATE_FAILED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  EditEntity ||--o{ TaskSubmitted : emits
  EditEntity ||--o{ AnalysisStarted : emits
  EditEntity ||--o{ EditProposed : emits
  EditEntity ||--o{ GatePassed : emits
  EditEntity ||--o{ GateFailed : emits
  EditEntity ||--o{ EditFailed : emits
  EditView }o--|| EditEntity : projects
  CIGate }o--|| EditEntity : subscribes
  EditWorkflow }o--|| EditEntity : reads-and-writes
  CodeAssistantAgent ||--o{ EditPlan : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `EditEndpoint` | `api/EditEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `EditEntity` | `application/EditEntity.java` (state in `domain/Edit.java`, events in `domain/EditEvent.java`) |
| `CIGate` | `application/CIGate.java` |
| `EditWorkflow` | `application/EditWorkflow.java` |
| `CodeAssistantAgent` | `application/CodeAssistantAgent.java` (tasks in `application/EditTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `EditView` | `application/EditView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `analyzeStep` 90 s (accommodates LLM latency plus multi-file attachment parsing), `awaitGateStep` 30 s (CI gate is in-process), `error` 5 s (Lesson 4). Default step recovery `maxRetries(2).failoverTo(EditWorkflow::error)`.
- **Idempotency**: every workflow uses `"edit-" + editId` as the workflow id; `EditEndpoint` mints `editId` before writing the entity event, so a retry of the POST with the same body (same `taskDescription` and `snapshotId`) resolves to the same entity and workflow without duplication.
- **One agent per task**: the AutonomousAgent instance id is `"assistant-" + editId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries at 4.
- **Guardrail-driven retry**: when `ToolCallGuardrail` blocks a tool call, the rejection is returned as a structured `tool-blocked` error to the agent loop. The loop counts toward `maxIterationsPerTask`; if the agent exhausts its budget trying blocked calls, the workflow's `analyzeStep` fails over to `error` and the entity transitions to `FAILED`.
- **CI gate is synchronous and deterministic**: `CIGate` runs the test suite in-process against an in-memory copy of the repository snapshot. No LLM call, no external service. The same edit plan always produces the same gate result for the same repository snapshot, keeping the single-agent guarantee honest.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. The CI gate runs against an in-memory copy of the repository; it does not write to disk and has nothing to roll back.
