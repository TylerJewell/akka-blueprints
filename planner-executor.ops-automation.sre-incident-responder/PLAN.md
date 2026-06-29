# PLAN — sre-incident-responder

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

  Cmd[IncidentCommanderAgent]:::agent
  Tel[TelemetryAgent]:::agent
  Run[RunbookAgent]:::agent
  Rem[RemediationAgent]:::agent

  WF[IncidentWorkflow]:::wf
  Inc[IncidentEntity]:::ese
  Appr[ApprovalEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[AlertQueue]:::ese
  View[IncidentView]:::view
  Consumer[AlertRequestConsumer]:::cons
  Sim[AlertSimulator]:::ta
  Stale[StaleIncidentMonitor]:::ta
  API[IncidentEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit alert| Queue
  API -->|approve/reject| Appr
  API -->|halt/resume| Ctrl
  Sim -.->|every 120s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|TRIAGE / INVESTIGATE_DECIDE / PROPOSE_REMEDIATION / COMPOSE_REPORT / SCORE_INCIDENT| Cmd
  WF -->|FETCH_METRICS / FETCH_LOGS / FETCH_TRACES| Tel
  WF -->|LOOKUP_RUNBOOK| Run
  WF -->|EXECUTE_ACTION| Rem
  WF -->|emit events| Inc
  WF -->|request / poll approval| Appr
  WF -->|poll| Ctrl
  Inc -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 60s| Inc
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as SRE
  participant API as IncidentEndpoint
  participant Q as AlertQueue
  participant C as AlertRequestConsumer
  participant W as IncidentWorkflow
  participant Cmd as IncidentCommanderAgent
  participant P as Probe Agent (Telemetry/Runbook)
  participant Rem as RemediationAgent
  participant Inc as IncidentEntity
  participant Appr as ApprovalEntity
  participant CTL as SystemControlEntity
  participant V as IncidentView

  U->>API: POST /api/incidents {description, severity}
  API->>Q: append AlertSubmitted
  API-->>U: 202 {incidentId}
  Q->>C: AlertSubmitted
  C->>W: start({incidentId, description})
  W->>Inc: emit IncidentCreated (TRIAGING)
  W->>Cmd: TRIAGE(description)
  Cmd-->>W: InvestigationLedger
  W->>Inc: emit IncidentTriaged, status INVESTIGATING
  loop until ProposeRemediation | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>Cmd: INVESTIGATE_DECIDE(ledgers)
    Cmd-->>W: ContinueInvestigation(ProbeDecision)
    W->>W: probeGuardrail.vet(decision)
    W->>P: runSingleTask(probe)
    P-->>W: ProbeResult
    W->>Inc: emit ProbeRecorded (ProbeEntry)
  end
  W->>Cmd: PROPOSE_REMEDIATION(ledgers)
  Cmd-->>W: RemediationAction
  W->>W: remediationGuardrail.vet(action)
  W->>Appr: requestApproval(incidentId, action), status AWAITING_APPROVAL
  W->>Inc: emit RemediationProposed
  U->>API: POST /api/incidents/{id}/approve {approved:true}
  API->>Appr: recordDecision
  Appr-->>W: ApprovalDecided(approved=true)
  W->>Rem: EXECUTE_ACTION(action)
  Rem-->>W: ActionOutcome
  W->>W: safetyEvaluator.evaluate(outcome)
  W->>Inc: emit RemediationExecuted, status REMEDIATING
  W->>Cmd: COMPOSE_REPORT(ledgers)
  Cmd-->>W: PostIncidentReport
  W->>Inc: emit IncidentMitigated, status MITIGATED
  W->>Cmd: SCORE_INCIDENT(ledgers, report)
  Cmd-->>W: EvalScore
  W->>Inc: emit EvalScoreRecorded
  Inc-->>V: project
  V-->>U: SSE update
```

## State machine — `IncidentEntity`

```mermaid
stateDiagram-v2
  [*] --> TRIAGING
  TRIAGING --> INVESTIGATING: IncidentTriaged
  INVESTIGATING --> INVESTIGATING: ProbeRecorded / ProbeBlocked / InvestigationReplanned
  INVESTIGATING --> AWAITING_APPROVAL: RemediationProposed
  AWAITING_APPROVAL --> REMEDIATING: ApprovalGranted
  AWAITING_APPROVAL --> INVESTIGATING: ApprovalRejected
  AWAITING_APPROVAL --> UNRESOLVED: ApprovalTimeout
  REMEDIATING --> MITIGATED: IncidentMitigated
  REMEDIATING --> HALTED: IncidentHaltedAutomatic
  REMEDIATING --> INVESTIGATING: PostRemediateReplan
  INVESTIGATING --> UNRESOLVED: IncidentUnresolved
  INVESTIGATING --> HALTED: IncidentHaltedOperator
  INVESTIGATING --> TIMED_OUT: IncidentTimedOut
  MITIGATED --> [*]
  UNRESOLVED --> [*]
  HALTED --> [*]
  TIMED_OUT --> [*]
```

## Entity model

```mermaid
erDiagram
  IncidentEntity ||--o{ IncidentCreated : emits
  IncidentEntity ||--o{ IncidentTriaged : emits
  IncidentEntity ||--o{ ProbeDispatched : emits
  IncidentEntity ||--o{ ProbeBlocked : emits
  IncidentEntity ||--o{ ProbeRecorded : emits
  IncidentEntity ||--o{ InvestigationReplanned : emits
  IncidentEntity ||--o{ RemediationProposed : emits
  IncidentEntity ||--o{ ApprovalGranted : emits
  IncidentEntity ||--o{ ApprovalRejected : emits
  IncidentEntity ||--o{ RemediationExecuted : emits
  IncidentEntity ||--o{ IncidentMitigated : emits
  IncidentEntity ||--o{ IncidentUnresolved : emits
  IncidentEntity ||--o{ IncidentHaltedAutomatic : emits
  IncidentEntity ||--o{ IncidentHaltedOperator : emits
  IncidentEntity ||--o{ IncidentTimedOut : emits
  IncidentEntity ||--o{ EvalScoreRecorded : emits
  IncidentView }o--|| IncidentEntity : projects
  ApprovalEntity ||--o{ ApprovalRequested : emits
  ApprovalEntity ||--o{ ApprovalDecided : emits
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  AlertQueue ||--o{ AlertSubmitted : emits
  AlertRequestConsumer }o--|| AlertQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `IncidentCommanderAgent` | `application/IncidentCommanderAgent.java` |
| `TelemetryAgent` | `application/TelemetryAgent.java` |
| `RunbookAgent` | `application/RunbookAgent.java` |
| `RemediationAgent` | `application/RemediationAgent.java` |
| `IncidentWorkflow` | `application/IncidentWorkflow.java` |
| `IncidentEntity` | `application/IncidentEntity.java` (state in `domain/Incident.java`, events in `domain/IncidentEvent.java`) |
| `ApprovalEntity` | `application/ApprovalEntity.java` |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `AlertQueue` | `application/AlertQueue.java` |
| `IncidentView` | `application/IncidentView.java` |
| `AlertRequestConsumer` | `application/AlertRequestConsumer.java` |
| `AlertSimulator` | `application/AlertSimulator.java` |
| `StaleIncidentMonitor` | `application/StaleIncidentMonitor.java` |
| `ProbeGuardrail` | `application/ProbeGuardrail.java` |
| `RemediationGuardrail` | `application/RemediationGuardrail.java` |
| `SafetyEvaluator` | `application/SafetyEvaluator.java` |
| `CommanderTasks` | `application/CommanderTasks.java` |
| `ProbeTasks` | `application/ProbeTasks.java` |
| `RemediationTasks` | `application/RemediationTasks.java` |
| `IncidentEndpoint` | `api/IncidentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `triageStep` 60 s, `proposeProbeStep` 45 s, `dispatchProbeStep` 90 s (covers telemetry and runbook calls), `investigateDecideStep` 45 s, `proposeRemediationStep` 60 s, `awaitApprovalStep` 30 minutes, `executeRemediationStep` 120 s, `composeReportStep` 90 s, `scoreIncidentStep` 60 s. Default recovery: `maxRetries(2).failoverTo(IncidentWorkflow::error)`.
- **Replan budget:** the commander may emit `ReplanInvestigation` at most twice in a row without a `ContinueInvestigation`; a third consecutive replan is treated as `FailInvestigation`.
- **Failure budget:** the commander may emit `ContinueInvestigation` on the same `(probeKind, target)` pair at most three times; a fourth attempt is treated as `FailInvestigation`.
- **Approval timeout:** `awaitApprovalStep` expires after 30 minutes; on expiry the workflow routes to `unresolvedStep` with reason `"approval-timeout"`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchProbeStep` lets the in-flight probe finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `IncidentEndpoint.submit` uses `(description, reportedBy)` over a 30 s window to deduplicate `POST /api/incidents`.
- **Stale detection:** `StaleIncidentMonitor` ticks every 60 s; incidents `INVESTIGATING` for > 10 minutes are marked `TIMED_OUT`. The workflow's `investigateDecideStep` checks the entity's status and exits if it reads `TIMED_OUT`.
- **Safety halt determinism:** `SafetyEvaluator.evaluate` is pure; same input always yields the same outcome, keeping `IncidentEntity` events deterministic and replayable.
