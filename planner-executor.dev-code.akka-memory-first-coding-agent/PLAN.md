# PLAN — akka-memory-first-coding-agent

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

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

  RP[ResearchPlannerAgent]:::agent
  CR[CodeReaderAgent]:::agent
  MW[MemoryWriterAgent]:::agent
  EP2[EditPlannerAgent]:::agent
  EX[EditExecutorAgent]:::agent

  RWF[ResearchWorkflow]:::wf
  EWF[EditWorkflow]:::wf

  Proj[ProjectEntity]:::ese
  Sess[EditSessionEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  View[ProjectView]:::view
  Cons[SessionRequestConsumer]:::cons
  Sim[ResearchSimulator]:::ta
  Mon[StaleSessionMonitor]:::ta
  API[ProjectEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|init| Proj
  API -->|edit| Proj
  API -->|halt/resume| Ctrl
  Sim -.->|once on startup| Proj
  Proj -.->|EditSessionRequested| Cons
  Cons -->|start workflow| EWF
  RWF -->|SCAN_CODEBASE| RP
  RWF -->|READ_FILE x N| CR
  RWF -->|SYNTHESISE_MEMORY| MW
  RWF -->|buildMemory / rewritePrompt| Proj
  EWF -->|PLAN_EDITS| EP2
  EWF -->|APPLY_EDIT| EX
  EWF -->|emit events| Sess
  EWF -->|poll| Ctrl
  Proj -.->|projects| View
  Sess -.->|projects| View
  API -->|query/SSE| View
  Mon -.->|every 30s| Sess
```

## Interaction sequence — J1 (init happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ProjectEndpoint
  participant P as ProjectEntity
  participant RWF as ResearchWorkflow
  participant RP as ResearchPlannerAgent
  participant CR as CodeReaderAgent
  participant MW as MemoryWriterAgent
  participant V as ProjectView

  U->>API: POST /api/projects/init {projectPath}
  API->>P: createProject
  API-->>U: 202 {projectId}
  P-->>RWF: start(projectId, projectPath)
  RWF->>P: emit ProjectCreated (RESEARCHING)
  RWF->>RP: SCAN_CODEBASE(index)
  RP-->>RWF: ResearchPlan
  RWF->>P: emit ResearchPlanReady
  loop for each file in plan.filesToRead
    RWF->>CR: READ_FILE(fileName, question)
    CR-->>RWF: FileInsight
  end
  RWF->>MW: SYNTHESISE_MEMORY(insights)
  MW-->>RWF: ProjectMemory (blocks + systemPrompt)
  RWF->>P: buildMemory(memory)
  RWF->>P: rewritePrompt(systemPrompt)
  P->>P: emit MemoryBuilt, SystemPromptRewritten (READY)
  P-->>V: project
  V-->>U: SSE update (READY, block count)
```

## Interaction sequence — J2 (edit path with guardrail)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ProjectEndpoint
  participant P as ProjectEntity
  participant C as SessionRequestConsumer
  participant EWF as EditWorkflow
  participant EP2 as EditPlannerAgent
  participant EX as EditExecutorAgent
  participant S as EditSessionEntity
  participant CTL as SystemControlEntity
  participant V as ProjectView

  U->>API: POST /api/projects/{id}/edit {instruction}
  API->>P: requestEdit(instruction)
  P->>P: emit EditSessionRequested
  P-->>C: EditSessionRequested
  C->>EWF: start({sessionId, projectId, instruction})
  EWF->>S: emit SessionCreated (PLANNING)
  EWF->>EP2: PLAN_EDITS(memory, instruction)
  EP2-->>EWF: PatchPlan
  EWF->>S: emit PatchPlanReady (APPLYING)
  loop for each FileEdit in plan.edits
    EWF->>CTL: get halt flag
    CTL-->>EWF: halted=false
    EWF->>EWF: EditGuardrail.vet(edit)
    EWF->>EX: APPLY_EDIT(edit)
    EX-->>EWF: PatchResult
    EWF->>EWF: FixtureTestRunner.run(projectId)
    EWF->>S: emit PatchApplied + TestsPassed
  end
  EWF->>S: emit SessionCompleted
  S-->>V: session
  V-->>U: SSE update (COMPLETED)
```

## State machine — `EditSessionEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> APPLYING: PatchPlanReady
  APPLYING --> APPLYING: PatchApplied / TestsPassed / PatchBlocked
  APPLYING --> COMPLETED: SessionCompleted
  APPLYING --> TESTS_FAILED: TestsFailed
  APPLYING --> FAILED: SessionFailed
  APPLYING --> HALTED: SessionHalted
  APPLYING --> TIMED_OUT: SessionTimedOut
  COMPLETED --> [*]
  TESTS_FAILED --> [*]
  FAILED --> [*]
  HALTED --> [*]
  TIMED_OUT --> [*]
```

## Entity model

```mermaid
erDiagram
  ProjectEntity ||--o{ ProjectCreated : emits
  ProjectEntity ||--o{ ResearchPlanReady : emits
  ProjectEntity ||--o{ MemoryBuilt : emits
  ProjectEntity ||--o{ SystemPromptRewritten : emits
  ProjectEntity ||--o{ ProjectFailed : emits
  ProjectEntity ||--o{ EditSessionRequested : emits
  EditSessionEntity ||--o{ SessionCreated : emits
  EditSessionEntity ||--o{ PatchPlanReady : emits
  EditSessionEntity ||--o{ PatchBlocked : emits
  EditSessionEntity ||--o{ PatchApplied : emits
  EditSessionEntity ||--o{ TestsPassed : emits
  EditSessionEntity ||--o{ TestsFailed : emits
  EditSessionEntity ||--o{ SessionCompleted : emits
  EditSessionEntity ||--o{ SessionFailed : emits
  EditSessionEntity ||--o{ SessionHalted : emits
  EditSessionEntity ||--o{ SessionTimedOut : emits
  ProjectView }o--|| ProjectEntity : projects
  ProjectView }o--|| EditSessionEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  SessionRequestConsumer }o--|| ProjectEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ResearchPlannerAgent` | `application/ResearchPlannerAgent.java` |
| `CodeReaderAgent` | `application/CodeReaderAgent.java` |
| `MemoryWriterAgent` | `application/MemoryWriterAgent.java` |
| `EditPlannerAgent` | `application/EditPlannerAgent.java` |
| `EditExecutorAgent` | `application/EditExecutorAgent.java` |
| `ResearchWorkflow` | `application/ResearchWorkflow.java` |
| `EditWorkflow` | `application/EditWorkflow.java` |
| `ProjectEntity` | `application/ProjectEntity.java` (state in `domain/Project.java`, events in `domain/ProjectEvent.java`) |
| `EditSessionEntity` | `application/EditSessionEntity.java` (state in `domain/EditSession.java`, events in `domain/SessionEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `ProjectView` | `application/ProjectView.java` |
| `SessionRequestConsumer` | `application/SessionRequestConsumer.java` |
| `ResearchSimulator` | `application/ResearchSimulator.java` |
| `StaleSessionMonitor` | `application/StaleSessionMonitor.java` |
| `EditGuardrail` | `application/EditGuardrail.java` |
| `FixtureTestRunner` | `application/FixtureTestRunner.java` |
| `ResearchTasks` | `application/ResearchTasks.java` |
| `EditTasks` | `application/EditTasks.java` |
| `ProjectEndpoint` | `api/ProjectEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `scanStep` 45 s, `readFileStep` 60 s (per iteration), `synthesiseStep` 90 s, `planStep` 60 s, `applyStep` 90 s, `testGateStep` 120 s, `decideStep` 30 s. Default recovery: `maxRetries(2).failoverTo(workflow::error)`.
- **Guardrail-replan budget:** at most two consecutive blocked patches before the session fails with `"guardrail blocked all revisions"`.
- **Test-gate is terminal:** a single test failure ends the session in `TESTS_FAILED`. The patch is never applied to the project's canonical memory.
- **Halt poll:** `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. A halt arriving during `applyStep` lets the apply finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `POST /api/projects/init` deduplicates on `(projectPath, requestedBy)` over a 30 s window.
- **Stuck detection:** `StaleSessionMonitor` ticks every 30 s; sessions in `APPLYING` for more than 5 minutes receive `SessionTimedOut`. The workflow's `decideStep` reads the session status and exits on `TIMED_OUT`.
- **Self-rewrite loading:** `MemoryWriterAgent`'s effective prompt is resolved in `synthesiseStep` — the workflow calls `ProjectEntity.getProject` first to obtain the current `memory.systemPrompt` (if non-empty) and passes it to the agent invocation as the overriding system prompt. This ensures the first run uses the static prompt file and all subsequent runs use the persisted rewrite.
