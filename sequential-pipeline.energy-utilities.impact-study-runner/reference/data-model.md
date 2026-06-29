# Data model — impact-study-runner

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BusReading` | `busId` | `String` | no | Short identifier for the bus (e.g. `BUS-101`). |
| | `busName` | `String` | no | Human-readable bus name and voltage level. |
| | `voltageKv` | `double` | no | Measured bus voltage in kilovolts. |
| | `voltagePu` | `double` | no | Voltage in per-unit (normalised to nominal). |
| | `measuredAt` | `Instant` | no | When the load-flow computation recorded this reading. |
| `LineFlow` | `lineId` | `String` | no | Short identifier for the transmission line. |
| | `fromBus` | `String` | no | `BusReading.busId` of the sending end. |
| | `toBus` | `String` | no | `BusReading.busId` of the receiving end. |
| | `flowMw` | `double` | no | Active power flow in megawatts. |
| | `loadPercent` | `double` | no | Flow as a percentage of the line's thermal rating. |
| | `measuredAt` | `Instant` | no | When the load-flow computation recorded this flow. |
| `LoadFlowResult` | `busReadings` | `List<BusReading>` | no | Possibly empty (J6 demonstrates). |
| | `lineFlows` | `List<LineFlow>` | no | Possibly empty. |
| | `computedAt` | `Instant` | no | When the LOAD_FLOW task returned. |
| `Violation` | `contingencyId` | `String` | no | Identifier for the contingency scenario (e.g. `CONT-LINE-201-OUT`). |
| | `elementName` | `String` | no | Name of the element that overloads under this contingency. |
| | `limitType` | `String` | no | `thermal-MW` or `voltage-kV`. |
| | `limit` | `double` | no | The applicable limit value. |
| | `observed` | `double` | no | The value observed under the contingency. |
| | `isBinding` | `boolean` | no | True when `observed > limit` beyond tolerance. |
| `VoltageCheck` | `busId` | `String` | no | MUST equal a `BusReading.busId` from the upstream `LoadFlowResult`. |
| | `voltageKv` | `double` | no | Bus voltage being checked. |
| | `lowerLimitKv` | `double` | no | Lower operating-voltage limit from `grid-params.json`. |
| | `upperLimitKv` | `double` | no | Upper operating-voltage limit from `grid-params.json`. |
| | `withinLimits` | `boolean` | no | True when `lowerLimitKv <= voltageKv <= upperLimitKv`. |
| `ContingencyResult` | `violations` | `List<Violation>` | no | Possibly empty. |
| | `voltageChecks` | `List<VoltageCheck>` | no | Possibly empty. |
| | `n1ContingenciesTested` | `int` | no | Count of contingency scenarios evaluated. |
| | `analyzedAt` | `Instant` | no | When the CONTINGENCY task returned. |
| `BusRef` | `busId` | `String` | no | MUST equal a `BusReading.busId` from the upstream `LoadFlowResult`. |
| | `busName` | `String` | no | Matching bus name. |
| `Section` | `findingId` | `String` | no | MUST equal a `Violation.contingencyId` from the upstream `ContingencyResult`. |
| | `heading` | `String` | no | Section heading derived from the violation's element name. |
| | `body` | `String` | no | 2–4 sentences describing the violation and its grid impact. |
| | `busRefs` | `List<BusRef>` | no | Non-empty; lists the affected buses. |
| `StudyReport` | `requestId` | `String` | no | The interconnection request ID. |
| | `executiveSummary` | `String` | no | 1-paragraph summary of N-1 compliance status. |
| | `sections` | `List<Section>` | no | One section per binding violation (`isBinding == true`). |
| | `meetsN1Criterion` | `boolean` | no | MUST be `false` if any `Violation.isBinding == true`. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `SimulatorScorer` finished. |
| `StudyRecord` (entity state) | `studyId` | `String` | no | — |
| | `requestId` | `Optional<String>` | yes | Populated after `StudyCreated`. |
| | `loadFlow` | `Optional<LoadFlowResult>` | yes | Populated after `LoadFlowComputed`. |
| | `contingency` | `Optional<ContingencyResult>` | yes | Populated after `ContingencyAnalyzed`. |
| | `report` | `Optional<StudyReport>` | yes | Populated after `ReportDrafted`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `approvalNote` | `Optional<String>` | yes | Populated after `StudyApproved` or `StudyRejected`. |
| | `status` | `StudyStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `StudyCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `StudyRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`StudyStatus`: `CREATED`, `RUNNING_LOAD_FLOW`, `LOAD_FLOW_DONE`, `RUNNING_CONTINGENCY`, `CONTINGENCY_DONE`, `DRAFTING`, `REPORT_DRAFTED`, `AWAITING_APPROVAL`, `APPROVED`, `REJECTED`, `FAILED`.

## Events (`StudyEntity`)

| Event | Payload | Transition |
|---|---|---|
| `StudyCreated` | `requestId: String` | → CREATED |
| `LoadFlowStarted` | — | → RUNNING_LOAD_FLOW |
| `LoadFlowComputed` | `loadFlow: LoadFlowResult` | → LOAD_FLOW_DONE |
| `ContingencyStarted` | — | → RUNNING_CONTINGENCY |
| `ContingencyAnalyzed` | `contingency: ContingencyResult` | → CONTINGENCY_DONE |
| `DraftStarted` | — | → DRAFTING |
| `ReportDrafted` | `report: StudyReport` | → REPORT_DRAFTED |
| `EvaluationScored` | `eval: EvalResult` | (status stays REPORT_DRAFTED; combined with ApprovalRequested → AWAITING_APPROVAL) |
| `ApprovalRequested` | — | → AWAITING_APPROVAL |
| `StudyApproved` | `note: Optional<String>` | → APPROVED (terminal happy) |
| `StudyRejected` | `reason: String` | → REJECTED (terminal) |
| `StudyFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `StudyRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`StudyRow` mirrors `StudyRecord` exactly. The UI fetches the full row via `GET /api/studies/{id}` and streams updates via `GET /api/studies/sse`.

The view declares ONE query: `getAllStudies: SELECT * AS studies FROM study_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`StudyTasks.java`)

```java
public final class StudyTasks {
  public static final Task<LoadFlowResult> RUN_LOAD_FLOW = Task
      .name("Run load flow")
      .description("Compute bus voltages and power flows for the interconnection request")
      .resultConformsTo(LoadFlowResult.class);

  public static final Task<ContingencyResult> RUN_CONTINGENCY = Task
      .name("Run contingency")
      .description("Identify N-1 violations and check voltage thresholds against grid parameters")
      .resultConformsTo(ContingencyResult.class);

  public static final Task<StudyReport> DRAFT_REPORT = Task
      .name("Draft report")
      .description("Compose a StudyReport whose Sections address each binding contingency violation one-to-one")
      .resultConformsTo(StudyReport.class);

  private StudyTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-scoped tools

| Tool | Phase | Class | Returns |
|---|---|---|---|
| `computeBusVoltages(requestId)` | LOAD_FLOW | `LoadFlowTools` | `List<BusReading>` |
| `computePowerFlows(requestId)` | LOAD_FLOW | `LoadFlowTools` | `List<LineFlow>` |
| `runN1Analysis(loadFlowResult)` | CONTINGENCY | `ContingencyTools` | `List<Violation>` |
| `checkVoltageThresholds(loadFlowResult)` | CONTINGENCY | `ContingencyTools` | `List<VoltageCheck>` |
| `formatFinding(violation, loadFlowResult)` | DRAFT | `DraftTools` | `Section` |
| `assembleExecutiveSummary(violations)` | DRAFT | `DraftTools` | `String` |

All six tools are registered on `StudyAgent`. Phase isolation is enforced by the workflow step's task scoping — each `runSingleTask` call carries `metadata("phase", "<PHASE>")` so the agent prompt can stay phase-disciplined, and the task's typed return type gates which tools the agent will use in practice.
