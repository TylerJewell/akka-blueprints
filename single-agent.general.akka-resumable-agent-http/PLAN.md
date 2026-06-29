# PLAN — akka-resumable-agent-http

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

  API[AgentEndpoint]:::ep
  Entity[AgentRunEntity]:::ese
  WF[AgentRunWorkflow]:::wf
  Agent[WeatherQueryAgent]:::agent
  Tool[SlowWeatherTool]:::agent
  Guard[ToolCallGuardrail]:::guard
  Eval[IncidentEvaluator]:::guard
  View[AgentRunView]:::view
  App[AppEndpoint]:::ep

  API -->|initiate| Entity
  API -->|start workflow| WF
  WF -->|initStep emit| Entity
  WF -->|toolCallStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|calls| Tool
  Tool -->|WeatherReport| Agent
  Agent -->|WeatherReport| WF
  WF -->|completeToolCall| Entity
  WF -->|reportStep| Entity
  WF -->|incidentStep on resume| Eval
  Eval -->|IncidentEvent| WF
  WF -->|recordIncident| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as AgentEndpoint
  participant E as AgentRunEntity
  participant W as AgentRunWorkflow
  participant A as WeatherQueryAgent
  participant G as ToolCallGuardrail
  participant T as SlowWeatherTool

  U->>API: POST /api/agent/run
  API->>E: initiate(request)
  E-->>API: { runId }
  API->>W: start(runId)
  W->>E: emit RunInitiated
  W->>A: runSingleTask(location + queryType)
  A->>G: before-tool-call(location, queryType)
  G-->>A: accept
  A->>T: call SlowWeatherTool (8 s delay)
  Note over T: process kill point for J2
  T-->>A: WeatherReport
  A-->>W: WeatherReport
  W->>E: completeToolCall(report)
  W->>E: generateReport + complete
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `AgentRunEntity`

```mermaid
stateDiagram-v2
  [*] --> INITIATED
  INITIATED --> TOOL_CALLED: ToolCallStarted
  TOOL_CALLED --> REPORTING: ToolCallCompleted
  REPORTING --> COMPLETED: RunCompleted
  TOOL_CALLED --> RESUMED: RunResumed (crash during tool call)
  REPORTING --> RESUMED: RunResumed (crash during report)
  RESUMED --> TOOL_CALLED: ToolCallStarted (retry from checkpoint)
  RESUMED --> REPORTING: ToolCallCompleted (resume after tool done)
  INITIATED --> FAILED: RunFailed
  TOOL_CALLED --> FAILED: RunFailed (guardrail-exhaustion)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  AgentRunEntity ||--o{ RunInitiated : emits
  AgentRunEntity ||--o{ ToolCallStarted : emits
  AgentRunEntity ||--o{ ToolCallCompleted : emits
  AgentRunEntity ||--o{ ReportGenerated : emits
  AgentRunEntity ||--o{ RunCompleted : emits
  AgentRunEntity ||--o{ RunFailed : emits
  AgentRunEntity ||--o{ RunResumed : emits
  AgentRunView }o--|| AgentRunEntity : projects
  AgentRunWorkflow }o--|| AgentRunEntity : reads-and-writes
  WeatherQueryAgent ||--o{ WeatherReport : returns
  IncidentEvaluator ||--o{ IncidentEvent : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AgentEndpoint` | `api/AgentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AgentRunEntity` | `application/AgentRunEntity.java` (state in `domain/AgentRun.java`, events in `domain/AgentRunEvent.java`) |
| `AgentRunWorkflow` | `application/AgentRunWorkflow.java` |
| `WeatherQueryAgent` | `application/WeatherQueryAgent.java` (tasks in `application/AgentTasks.java`) |
| `SlowWeatherTool` | `application/SlowWeatherTool.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `IncidentEvaluator` | `application/IncidentEvaluator.java` |
| `AgentRunView` | `application/AgentRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `initStep` 5 s, `toolCallStep` 120 s, `reportStep` 30 s, `incidentStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AgentRunWorkflow::error)`. The 120 s on `toolCallStep` accommodates the configured tool delay plus LLM round-trip (Lesson 4).
- **Crash-resume checkpoint**: Akka Workflow persists each step's completion boundary. On restart the runtime replays from the last persisted step. The key invariant: once `toolCallStep` lands `ToolCallCompleted`, a crash in `reportStep` resumes at `reportStep`, not at `toolCallStep`. The agent and tool are NOT re-invoked.
- **Resume detection**: on `AgentRunWorkflow.onInit`, if `AgentRunEntity.getRun()` shows `status == TOOL_CALLED` or `status == REPORTING`, the workflow emits `RunResumed` and increments `resumeCount` before proceeding. This makes the resume observable without any external coordinator.
- **Idempotency**: workflow id is `"run-" + runId`; starting the workflow twice for the same `runId` is a no-op. `AgentRunEntity` commands are event-version-guarded so duplicate deliveries do not produce duplicate events.
- **Guardrail-driven retry**: when `ToolCallGuardrail` rejects a tool invocation, the agent loop consumes one iteration and retries the tool call with corrected arguments. If all 3 iterations fail, `toolCallStep` fails over to `error` and the entity transitions to `FAILED`.
- **One agent per run**: the AutonomousAgent instance id is `"agent-" + runId`, giving each run its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **No saga / no compensation**: every step is append-only. The `SlowWeatherTool` is idempotent (same inputs return deterministically ordered mock outputs) so retry after crash is safe.
