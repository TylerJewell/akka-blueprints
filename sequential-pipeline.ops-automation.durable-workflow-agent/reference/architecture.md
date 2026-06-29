# Architecture — durable-workflow-backed-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on one agent that runs three tasks in sequence, backed by a durable workflow that guarantees each phase completes even under transient failures. `RemediationEndpoint` accepts a `{alertId, serviceId}` POST, writes `IncidentCreated` onto `IncidentEntity`, and starts `RemediationWorkflow` keyed by `"remediation-" + incidentId`. The workflow's first step (`diagnoseStep`) emits `DiagnoseStarted`, then calls `OpsAgent` with `TaskDef.taskType(DIAGNOSE_INCIDENT)` and a `phase = DIAGNOSE` metadata tag. The agent invokes `DiagnoseTools.queryMetrics` and `DiagnoseTools.fetchLogs`; every call passes through `BudgetGuardrail` first. Once the agent returns a `DiagnosisReport`, the workflow writes `DiagnosisCompleted` onto the entity and advances to `remediateStep` — same pattern, the REMEDIATE task carries `phase = REMEDIATE`. Then `verifyStep` runs with `phase = VERIFY`. After `VerificationCompleted` lands, the incident is in its terminal happy state. `IncidentView` projects every event into a read-model row; `RemediationEndpoint` serves the read model to the UI over REST and SSE.

Separately, `DurabilityTimerAction` registers an hourly timer that fires `DurabilityEvaluator`. The evaluator queries `IncidentView` for incidents closed in the trailing window, computes four durability metrics, and writes a `DurabilityReport` to `DurabilityReportEntity`. The App UI Overview panel renders the latest report.

The graph has exactly one LLM-calling component. `DurabilityEvaluator` is a deterministic rule-based aggregator; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `diagnoseStep` and `remediateStep`, the workflow writes `DiagnosisCompleted` onto the entity. The next step reads the recorded `DiagnosisReport` from the entity to build the REMEDIATE task's instruction context. The agent never sees diagnose-phase context inside the remediate task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `BudgetGuardrail`. The guardrail counts the call against the per-phase cap and halts the agent when the cap is reached. A budget-cap halt fires before the tool body executes.

The agent calls are bounded by per-step timeouts (90 s on diagnose and remediate, 60 s on verify). The workflow's default step recovery (`maxRetries(2).failoverTo(error)`) provides the durability guarantee: if a step times out or the agent returns an undeserializable result, the step is retried up to twice before the incident transitions to `FAILED`.

## State machine

Eight states. The interesting paths:

- The happy path walks `CREATED → DIAGNOSING → DIAGNOSED → REMEDIATING → REMEDIATED → VERIFYING → VERIFIED`.
- Three failure transitions land in `FAILED`: an exhausted step-recovery budget during `DIAGNOSING`, `REMEDIATING`, or `VERIFYING`.
- `BudgetExhausted` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `ESCALATED` or `APPROVED` state. The incident's resolution is confirmed by the agent's `VerificationResult`; the ops team monitors the `DurabilityReport` for aggregate trends. The blueprint deliberately stops at `VERIFIED`.

## Entity model

`IncidentEntity` is the source of truth. It emits nine event types — three lifecycle starts, three lifecycle completions, the budget audit, the failure, and the initial creation. `IncidentView` projects every event into a row used by the UI and by `DurabilityEvaluator`. `RemediationWorkflow` both reads (`getIncident`) and writes (`startDiagnose`, `recordDiagnosis`, `startRemediate`, `recordRemediation`, `startVerify`, `recordVerification`, `recordBudgetExhausted`, `fail`) on the entity. `DurabilityReportEntity` is a separate entity that accumulates report events from `DurabilityEvaluator`.

## Defence-in-depth governance flow

For any incident that lands in the entity log, the alertId passed through:

1. **Budget-cap guardrail** — every tool call is filtered. A task that would issue its ninth call in the DIAGNOSE phase (cap: 8) is halted before the tool body runs; a `BudgetExhausted` event records the violation for audit.
2. **OpsAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase. The workflow's step timeouts and retry policy are the durability guarantee around each call.
3. **Scheduled durability evaluator** — every hour, a rolling window of closed incidents is evaluated for completion rate, budget-violation rate, MTTR, and timeout rate. The result is surfaced in the Overview panel, giving the ops team the aggregate view that no single incident's lifecycle can provide.

Each layer is independent. The guardrail does not compute metrics; the evaluator does not enforce phase order. Removing either one opens an explicit gap the other does not silently cover.
