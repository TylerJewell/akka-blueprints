# PLAN — Parallelization Workflow (Sectioning + Voting)

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  JE[JobEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  JQ[JobQueue<br/>EventSourcedEntity]:::ese
  JC[JobRequestConsumer<br/>Consumer]:::con
  WF[ParallelWorkflow<br/>Workflow]:::wf
  PS[ParallelSupervisor<br/>AutonomousAgent]:::ag
  SW[SectionWorker<br/>AutonomousAgent]:::ag
  VW[VoteWorker<br/>AutonomousAgent]:::ag
  JEnt[JobEntity<br/>EventSourcedEntity]:::ese
  JV[JobView<br/>View]:::vw
  SIM[JobSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  JE -->|POST /jobs| JQ
  SIM -.->|every 60s| JQ
  JQ -.->|JobSubmitted| JC
  JC -->|start workflow| WF
  WF -->|PARTITION| PS
  WF -->|PROCESS_SECTION ×N| SW
  WF -->|CAST_VOTE ×N| VW
  WF -->|AGGREGATE| PS
  WF -->|commands| JEnt
  JEnt -.->|events| JV
  EV -.->|every 5m| JV
  EV -->|recordEval| JEnt
  JE -->|getAllJobs / SSE| JV
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

Solid arrows: synchronous commands. Dashed arrows: event subscriptions / scheduler ticks. Worker fan-out multiplicity (×N) depends on the job's `workerCount` field, set by the Supervisor during PARTITION.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant JE as JobEndpoint
  participant JQ as JobQueue
  participant WF as ParallelWorkflow
  participant PS as ParallelSupervisor
  participant SW as SectionWorker
  participant VW as VoteWorker
  participant JEnt as JobEntity

  U->>JE: POST /api/jobs {prompt, mode}
  JE->>JQ: enqueueJob
  JQ-->>WF: JobRequestConsumer starts workflow
  WF->>JEnt: createJob (PARTITIONING)
  WF->>PS: PARTITION -> Partition or VotePlan
  WF->>JEnt: dispatchWorkers (IN_PROGRESS)
  alt SECTIONING mode
    par parallel fan-out
      WF->>SW: PROCESS_SECTION(section 0)
    and
      WF->>SW: PROCESS_SECTION(section 1)
    and
      WF->>SW: PROCESS_SECTION(section N-1)
    end
  else VOTING mode
    par parallel fan-out
      WF->>VW: CAST_VOTE(worker 0)
    and
      WF->>VW: CAST_VOTE(worker 1)
    and
      WF->>VW: CAST_VOTE(worker N-1)
    end
  end
  Note over WF: join all; if any step times out (60s) -> partialStep
  WF->>PS: AGGREGATE(all results) -> AggregatedOutput
  WF->>WF: guardrailStep vets aggregated output
  alt guardrail passes
    WF->>JEnt: aggregate (AGGREGATED)
  else guardrail fails
    WF->>JEnt: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PARTITIONING
  PARTITIONING --> IN_PROGRESS: workers dispatched
  IN_PROGRESS --> AGGREGATED: all results received + guardrail pass
  IN_PROGRESS --> PARTIAL: a worker timed out
  IN_PROGRESS --> BLOCKED: guardrail fail
  PARTIAL --> [*]
  BLOCKED --> [*]
  AGGREGATED --> AGGREGATED: EvalScored
  AGGREGATED --> [*]
```

## Entity model

```mermaid
erDiagram
  JOB ||--o{ SECTION_RESULT : "has (SECTIONING)"
  JOB ||--o{ VOTE_RESULT : "has (VOTING)"
  JOB ||--o| AGGREGATED_OUTPUT : produces
  JOB_QUEUE ||--|| JOB : seeds
  JOB {
    string jobId
    string prompt
    enum mode
    enum status
    int workerCount
    int evalScore
    instant createdAt
  }
  JOB_QUEUE {
    string jobId
    string prompt
    enum mode
    string submittedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `ParallelSupervisor` | AutonomousAgent | `application/ParallelSupervisor.java` |
| `SectionWorker` | AutonomousAgent | `application/SectionWorker.java` |
| `VoteWorker` | AutonomousAgent | `application/VoteWorker.java` |
| `ParallelTasks` | Task constants | `application/ParallelTasks.java` |
| `ParallelWorkflow` | Workflow | `application/ParallelWorkflow.java` |
| `JobEntity` | EventSourcedEntity | `domain/JobEntity.java` |
| `JobQueue` | EventSourcedEntity | `domain/JobQueue.java` |
| `JobView` | View | `application/JobView.java` |
| `JobRequestConsumer` | Consumer | `application/JobRequestConsumer.java` |
| `JobSimulator` | TimedAction | `application/JobSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `JobEndpoint` | HttpEndpoint | `api/JobEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** each worker step (PROCESS_SECTION, CAST_VOTE) gets 60s; aggregateStep gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** all worker calls for a given job run concurrently via `CompletionStage` allOf / zip. The number of concurrent workers is set by `Partition.workerCount` (SECTIONING) or `VotePlan.workerCount` (VOTING).
- **Idempotency:** the workflow id is the `jobId`. Re-delivery of the same `JobSubmitted` event resolves to the same workflow instance — no duplicate job.
- **Partial path (compensation):** if any worker times out, `defaultStepRecovery` routes to `partialStep`, which aggregates from whatever results arrived and ends with `JobPartial`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `JobView.getAllJobs` (no enum WHERE clause) and filters client-side for the oldest `AGGREGATED` job lacking an `evalScore`.
