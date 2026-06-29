# PLAN — autonomous-agent-pattern

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
  classDef helper fill:#181818,stroke:#888,color:#aaa;

  API[AgentTaskEndpoint]:::ep
  Entity[AgentTaskEntity]:::ese
  WF[AgentTaskWorkflow]:::wf
  Agent[CodingAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  Monitor[RuntimeMonitor]:::cons
  Halt[HaltChecker]:::helper
  View[AgentTaskView]:::view
  App[AppEndpoint]:::ep
  GH[GitHub API]:::helper

  API -->|submit| Entity
  API -->|start workflow| WF
  API -->|approve/reject/halt| Entity
  WF -->|fetchIssueStep| GH
  GH -->|IssueBody| WF
  WF -->|recordIssueFetched| Entity
  WF -->|planStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|AgentPlan| WF
  WF -->|recordPlan / checkpointPending| Entity
  WF -->|checkpointStep poll| Entity
  WF -->|executeStep runSingleTask| Agent
  Halt -.->|check haltRequested| Entity
  Agent -->|TaskOutcome| WF
  WF -->|recordOutcome| Entity
  Entity -.->|ToolCallExecuted| Monitor
  Monitor -->|raiseMonitorAlert| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as AgentTaskEndpoint
  participant E as AgentTaskEntity
  participant W as AgentTaskWorkflow
  participant GH as GitHub API
  participant A as CodingAgent
  participant G as ToolCallGuardrail
  participant M as RuntimeMonitor

  U->>API: POST /api/tasks
  API->>E: submit(issueRef)
  E-->>API: { taskId }
  API->>W: start(taskId)
  W->>GH: GET /repos/{owner}/{repo}/issues/{n}
  GH-->>W: IssueBody
  W->>E: recordIssueFetched(issueBody)
  W->>A: runSingleTask PLAN_ISSUE (issue attachment)
  A-->>W: AgentPlan
  W->>E: recordPlan + checkpointPending
  E-.->>U: SSE event(CHECKPOINT_PENDING)
  U->>API: POST /api/tasks/{id}/approve
  API->>E: approveCheckpoint
  W->>E: poll approved
  W->>A: runSingleTask RESOLVE_ISSUE (issue + plan attachments)
  A->>G: before-tool-call(readFile)
  G-->>A: allow
  A->>G: before-tool-call(writeFile)
  G-->>A: allow
  A-->>W: TaskOutcome
  W->>E: recordOutcome
  E-.->>M: ToolCallExecuted × N
  M->>E: (no alert — within rate)
  E-.->>U: SSE event(PATCH_READY)
```

## State machine — `AgentTaskEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> ISSUE_FETCHED: IssueFetched
  ISSUE_FETCHED --> PLAN_READY: PlanProposed
  PLAN_READY --> CHECKPOINT_PENDING: CheckpointPending
  CHECKPOINT_PENDING --> EXECUTING: CheckpointApproved
  CHECKPOINT_PENDING --> FAILED: CheckpointRejected
  EXECUTING --> PATCH_READY: OutcomeRecorded
  EXECUTING --> HALTED: TaskHalted
  EXECUTING --> FAILED: TaskFailed (agent/tool error)
  SUBMITTED --> FAILED: TaskFailed (fetch error)
  ISSUE_FETCHED --> FAILED: TaskFailed (plan error)
  PATCH_READY --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  AgentTaskEntity ||--o{ TaskSubmitted : emits
  AgentTaskEntity ||--o{ IssueFetched : emits
  AgentTaskEntity ||--o{ PlanProposed : emits
  AgentTaskEntity ||--o{ CheckpointPending : emits
  AgentTaskEntity ||--o{ CheckpointApproved : emits
  AgentTaskEntity ||--o{ CheckpointRejected : emits
  AgentTaskEntity ||--o{ ExecutionStarted : emits
  AgentTaskEntity ||--o{ ToolCallExecuted : emits
  AgentTaskEntity ||--o{ OutcomeRecorded : emits
  AgentTaskEntity ||--o{ HaltRequested : emits
  AgentTaskEntity ||--o{ TaskHalted : emits
  AgentTaskEntity ||--o{ TaskFailed : emits
  AgentTaskEntity ||--o{ MonitorAlertRaised : emits
  AgentTaskView }o--|| AgentTaskEntity : projects
  RuntimeMonitor }o--|| AgentTaskEntity : subscribes
  AgentTaskWorkflow }o--|| AgentTaskEntity : reads-and-writes
  CodingAgent ||--o{ TaskOutcome : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AgentTaskEndpoint` | `api/AgentTaskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AgentTaskEntity` | `application/AgentTaskEntity.java` (state in `domain/AgentTask.java`, events in `domain/AgentTaskEvent.java`) |
| `AgentTaskWorkflow` | `application/AgentTaskWorkflow.java` |
| `CodingAgent` | `application/CodingAgent.java` (tasks in `application/AgentTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `HaltChecker` | `application/HaltChecker.java` |
| `RuntimeMonitor` | `application/RuntimeMonitor.java` |
| `AgentTaskView` | `application/AgentTaskView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `fetchIssueStep` 10 s, `planStep` 60 s, `checkpointStep` 305 s, `executeStep` 300 s, `outcomeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(AgentTaskWorkflow::error)` (Lesson 4). The 300 s on `executeStep` covers multi-iteration tool loops.
- **Idempotency**: every workflow uses `"task-" + taskId` as the workflow id; `AgentTaskEntity.submit` is idempotent on a second call with the same `taskId` — it returns the existing `taskId` without emitting a second `TaskSubmitted`.
- **One agent per task**: the CodingAgent instance id is `"planner-" + taskId` for the plan phase and `"executor-" + taskId` for the execute phase, giving each phase its own conversation context. The agent's `capability(...).maxIterationsPerTask(12)` bounds the tool loop.
- **Guardrail-driven recovery**: when `ToolCallGuardrail` blocks a call, the block reason is returned to the agent as a structured `tool-blocked` error. The agent loop proposes an alternative within the same iteration budget. Blocked calls do not count toward `maxIterationsPerTask` — only full iterations do.
- **Halt is collaborative**: the halt flag on `AgentTaskEntity` is the source of truth. `HaltChecker` reads it at the start of each iteration. If the current iteration is already in a tool call, that call completes before the halt takes effect. No JVM thread interrupts.
- **Monitor is non-blocking**: `RuntimeMonitor` Consumer processes `ToolCallExecuted` events asynchronously. An alert never delays the agent loop. If `RuntimeMonitor` itself fails, the agent continues — monitoring is best-effort.
- **No external writes before checkpoint**: `checkpointStep` (step 3 of 5) gates all `writeFile` and `runShell` tool calls. The `planStep` agent call is declared with `TaskDef.instructions("Phase: PLAN — produce an AgentPlan only, no writes.")` — the system prompt and task instruction together prevent writes in the plan phase.
