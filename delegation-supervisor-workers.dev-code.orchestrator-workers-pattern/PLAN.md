# PLAN — Orchestrator-Workers Workflow

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  EE[EditEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  JQ[EditJobQueue<br/>EventSourcedEntity]:::ese
  JC[EditJobConsumer<br/>Consumer]:::con
  WF[EditWorkflow<br/>Workflow]:::wf
  OR[Orchestrator<br/>AutonomousAgent]:::ag
  FE[FileEditor<br/>AutonomousAgent]:::ag
  ER[EditReviewer<br/>AutonomousAgent]:::ag
  ET[EditTaskEntity<br/>EventSourcedEntity]:::ese
  TV[EditTaskView<br/>View]:::vw
  SIM[TaskSimulator<br/>TimedAction]:::ta
  QS[QualitySampler<br/>TimedAction]:::ta

  EE -->|POST /tasks| JQ
  SIM -.->|every 60s| JQ
  JQ -.->|TaskSubmitted| JC
  JC -->|start workflow| WF
  WF -->|DECOMPOSE| OR
  WF -->|EDIT per file| FE
  WF -->|REVIEW per file| ER
  WF -->|SYNTHESISE| OR
  WF -->|commands| ET
  ET -.->|events| TV
  QS -.->|every 5m| TV
  QS -->|recordQuality| ET
  EE -->|getAllTasks / SSE| TV
  AE --> STATIC[static-resources]:::static

  classDef ep fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ese fill:#141414,stroke:#F5C518,color:#fff;
  classDef vw fill:#141414,stroke:#3fb950,color:#fff;
  classDef wf fill:#141414,stroke:#ff5f57,color:#fff;
  classDef ag fill:#141414,stroke:#B388FF,color:#fff;
  classDef con fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ta fill:#141414,stroke:#F5C518,color:#fff;
  classDef static fill:#0A0A0A,stroke:#333,color:#aaa;
```

Solid arrows: synchronous commands. Dashed arrows: event subscriptions.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant EE as EditEndpoint
  participant JQ as EditJobQueue
  participant WF as EditWorkflow
  participant OR as Orchestrator
  participant FE as FileEditor
  participant ER as EditReviewer
  participant ET as EditTaskEntity

  U->>EE: POST /api/tasks {description, targetFiles}
  EE->>JQ: submitTask
  JQ-->>WF: EditJobConsumer starts workflow
  WF->>ET: createTask (PLANNING)
  WF->>OR: DECOMPOSE -> DecompositionPlan
  WF->>ET: recordPlan, status IN_PROGRESS
  par parallel per file
    WF->>FE: EDIT(file1) [guardrail gates applyEdit]
    WF->>FE: EDIT(file2) [guardrail gates applyEdit]
  end
  Note over WF: if any editStep times out (60s) -> degradeStep
  par parallel reviews
    WF->>ER: REVIEW(editedFile1)
    WF->>ER: REVIEW(editedFile2)
  end
  WF->>OR: SYNTHESISE(editedFiles, reviews) -> Changeset
  alt all approved
    WF->>ET: complete (COMPLETED)
  else blocked path
    WF->>ET: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> IN_PROGRESS: DecompositionPlan ready
  IN_PROGRESS --> COMPLETED: all edits synthesised + guardrail pass
  IN_PROGRESS --> DEGRADED: a worker timed out
  IN_PROGRESS --> BLOCKED: guardrail rejected a tool call
  DEGRADED --> [*]
  BLOCKED --> [*]
  COMPLETED --> COMPLETED: QualityScored
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  EDIT_TASK ||--o{ DECOMPOSITION_PLAN : has
  EDIT_TASK ||--o{ EDITED_FILE : contains
  EDIT_TASK ||--o{ REVIEW_VERDICT : contains
  EDIT_TASK ||--o| CHANGESET : produces
  EDIT_JOB_QUEUE ||--|| EDIT_TASK : seeds
  EDIT_TASK {
    string taskId
    string description
    list targetFiles
    enum status
    int qualityScore
    instant createdAt
  }
  EDIT_JOB_QUEUE {
    string taskId
    string description
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `Orchestrator` | AutonomousAgent | `application/Orchestrator.java` |
| `FileEditor` | AutonomousAgent | `application/FileEditor.java` |
| `EditReviewer` | AutonomousAgent | `application/EditReviewer.java` |
| `EditTasks` | Task constants | `application/EditTasks.java` |
| `EditWorkflow` | Workflow | `application/EditWorkflow.java` |
| `EditTaskEntity` | EventSourcedEntity | `domain/EditTaskEntity.java` |
| `EditJobQueue` | EventSourcedEntity | `domain/EditJobQueue.java` |
| `EditTaskView` | View | `application/EditTaskView.java` |
| `EditJobConsumer` | Consumer | `application/EditJobConsumer.java` |
| `TaskSimulator` | TimedAction | `application/TaskSimulator.java` |
| `QualitySampler` | TimedAction | `application/QualitySampler.java` |
| `EditEndpoint` | HttpEndpoint | `api/EditEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** each `editStep` gets 60 s; each `reviewStep` gets 30 s; `synthesiseStep` gets 90 s. The 5 s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `editStep(i)` instances run concurrently across all target files via `CompletionStage` allOf, not sequential calls. `reviewStep(i)` follows the same pattern.
- **Before-tool-call guardrail:** every `applyEdit` invocation inside `FileEditor` is intercepted; the guardrail checks `filePath` against the task's declared `targetFiles` list before the call is forwarded to the tool.
- **Idempotency:** the workflow id is the `taskId`. Re-delivery of the same `TaskSubmitted` event resolves to the same workflow instance — no duplicate task.
- **Degrade path:** if any `editStep` times out, `defaultStepRecovery` routes to `degradeStep`, which synthesises from completed files and ends with `TaskDegraded`. No infinite retry.
- **Quality sampling:** `QualitySampler` reads `EditTaskView.getAllTasks` and filters client-side for the oldest `COMPLETED` task lacking a `qualityScore`.
