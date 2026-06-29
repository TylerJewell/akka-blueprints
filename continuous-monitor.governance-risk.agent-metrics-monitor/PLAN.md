# PLAN — agent-metrics-monitor

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

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

  Poller[MetricsPoller]:::ta
  Queue[RawMetricsQueue]:::ese
  Ingestor[MetricsIngestor]:::cons
  Detector[AnomalyDetectorAgent]:::agent
  Narrator[SummaryNarratorAgent]:::agent
  Evaluator[SummaryEvalAgent]:::agent
  WF[MetricsScanWorkflow]:::wf
  Entity[AgentMetricsEntity]:::ese
  View[MetricsView]:::view
  EvalRunner[EvalRunner]:::ta
  API[MetricsEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 60s| Queue
  Queue -.->|subscribes| Ingestor
  Ingestor -->|emit MetricSampleRecorded| Entity
  Ingestor -.->|start per batch| WF
  WF -->|call per agent| Detector
  WF -->|call once per batch| Narrator
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|query/SSE| View
  API -->|get detail| Entity
  EvalRunner -.->|every 30m| Entity
  EvalRunner -->|call per summary| Evaluator
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as MetricsPoller
  participant Q as RawMetricsQueue
  participant I as MetricsIngestor
  participant E as AgentMetricsEntity
  participant W as MetricsScanWorkflow
  participant D as AnomalyDetectorAgent
  participant N as SummaryNarratorAgent
  participant U as User (UI)
  participant API as MetricsEndpoint

  P->>Q: emit BatchReceived
  Q->>I: BatchReceived
  I->>E: emit MetricSampleRecorded (per row)
  I->>W: start({batchId, samples})
  W->>D: detect(agentId, samples) [per agent]
  D-->>W: AnomalyResult
  W->>N: narrate(batchId, results)
  N-->>W: HealthSummary
  W->>E: emit AnomalyDetected + SummaryRecorded (per agent)
  Note over E,U: MetricsView updated via projection
  U->>API: GET /api/metrics/agents
  API-->>U: AgentHealthRow[] (with currentStatus)
  U->>API: GET /api/metrics/agents/{id}
  API-->>U: AgentHealthState (full detail + narrative)
```

## State machine — `AgentMetricsEntity`

```mermaid
stateDiagram-v2
  [*] --> HEALTHY : first MetricSampleRecorded
  HEALTHY --> HEALTHY : normal batch (AnomalyDetected HEALTHY)
  HEALTHY --> DEGRADED : AnomalyDetected DEGRADED
  HEALTHY --> CRITICAL : AnomalyDetected CRITICAL
  DEGRADED --> HEALTHY : AnomalyDetected HEALTHY
  DEGRADED --> CRITICAL : AnomalyDetected CRITICAL
  CRITICAL --> DEGRADED : AnomalyDetected DEGRADED
  CRITICAL --> HEALTHY : AnomalyDetected HEALTHY
  HEALTHY --> HEALTHY : SummaryRecorded
  DEGRADED --> DEGRADED : SummaryRecorded
  CRITICAL --> CRITICAL : SummaryRecorded
  HEALTHY --> HEALTHY : EvalScored
  DEGRADED --> DEGRADED : EvalScored
  CRITICAL --> CRITICAL : EvalScored
```

## Entity model

```mermaid
erDiagram
  AgentMetricsEntity ||--o{ MetricSampleRecorded : emits
  AgentMetricsEntity ||--o{ AnomalyDetected : emits
  AgentMetricsEntity ||--o{ SummaryRecorded : emits
  AgentMetricsEntity ||--o{ EvalScored : emits
  MetricsView }o--|| AgentMetricsEntity : projects
  RawMetricsQueue ||--o{ BatchReceived : emits
  MetricsIngestor }o--|| RawMetricsQueue : subscribes
  MetricsScanWorkflow }o--|| AgentMetricsEntity : updates
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MetricsPoller` | `application/MetricsPoller.java` |
| `RawMetricsQueue` | `application/RawMetricsQueue.java` |
| `MetricsIngestor` | `application/MetricsIngestor.java` |
| `AnomalyDetectorAgent` | `application/AnomalyDetectorAgent.java` |
| `SummaryNarratorAgent` | `application/SummaryNarratorAgent.java` |
| `SummaryEvalAgent` | `application/SummaryEvalAgent.java` |
| `MetricsScanWorkflow` | `application/MetricsScanWorkflow.java` |
| `AgentMetricsEntity` | `application/AgentMetricsEntity.java` (state in `domain/AgentHealthState.java`, events in `domain/AgentMetricsEvent.java`) |
| `MetricsView` | `application/MetricsView.java` |
| `EvalRunner` | `application/EvalRunner.java` |
| `MetricsEndpoint` | `api/MetricsEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `detectStep` 20 s per agent, `narrateStep` 15 s. On timeout, record `AnomalyResult` with status `DEGRADED` and a signal `"detection-timeout"`.
- **Fan-out in detectStep**: the workflow calls `AnomalyDetectorAgent` once per agent in the batch; results are collected into a `List<AnomalyResult>` before `narrateStep` begins.
- **Idempotency**: every workflow uses `batchId` as the workflow id so duplicate `BatchReceived` events fold into one workflow run.
- **Sample cap**: `AgentMetricsEntity` retains only the 10 most recent `AgentMetricSample` records; older ones are dropped from in-memory state (the full audit is in `RawMetricsQueue`).
- **Eval sampling**: per tick, `EvalRunner` picks up to 5 agents with an un-scored `latestSummary`, oldest `lastUpdatedAt` first.
