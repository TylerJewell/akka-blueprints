# SPEC — field-service-dispatcher

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Scheduling Operations Agent.
**One-line pitch:** Submit a work order; a DispatcherAgent plans technician assignments on a schedule ledger, routes each job to a RouteOptimizerAgent or AvailabilityAgent, records outcomes on a fairness ledger, and replans when assignments fail or technicians become unavailable.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with two specialist executors sharing an ops-automation domain. The DispatcherAgent owns two ledgers — a **schedule ledger** (known work orders, unassigned slots, assignment plan, current dispatch) and a **fairness ledger** (per-technician assignment counts, cumulative travel distance, last-assigned timestamp). Each loop iteration the DispatcherAgent reads both ledgers, picks which specialist handles the next assignment step, and either continues, replans, pauses, or completes.

The blueprint wires two governance mechanisms to that loop:

- a **before-tool-call guardrail** that checks every proposed assignment against shift-window constraints, technician qualifications, and geographic feasibility before the specialist executes it,
- a **periodic fairness watch** that runs after each completed schedule and flags technician overloading or systematic under-assignment, writing alerts to the fairness ledger for the operator dashboard.

Two additional controls protect runtime integrity:

- an **operator pause control** that lets a human stop new dispatches without killing an in-flight assignment,
- a **credential sanitizer** that scrubs API-key-shaped and credential-shaped strings from every specialist output before it lands on the schedule ledger.

## 3. User-facing flows

The user opens the App UI tab and submits a work order via the form.

1. The system creates a `Schedule` record in `PLANNING` and starts a `ScheduleWorkflow`.
2. The DispatcherAgent drafts a `ScheduleLedger { knownOrders, unassignedSlots, plan, currentDispatch }` and emits `SchedulePlanned`.
3. The workflow enters the executor loop. Each iteration:
   - DispatcherAgent reads both ledgers and proposes an `AssignmentDecision { specialist, assignment, rationale }`.
   - The **before-tool-call guardrail** checks the decision; on rejection the workflow records an `AssignmentBlocked` entry on the fairness ledger and asks the DispatcherAgent to revise.
   - The chosen specialist runs the assignment step and returns a typed `AssignmentResult`.
   - The **credential sanitizer** scrubs the result.
   - The workflow appends an `AssignmentEntry { specialist, assignment, attempt, verdict, scrubbedResult }` to the fairness ledger.
   - The workflow checks the operator pause flag; on paused=true it finishes the in-flight step, then ends with `SchedulePausedOperator`.
   - The DispatcherAgent decides on each tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After three consecutive failures on the same assignment or two consecutive replans without progress, the DispatcherAgent emits `FAIL`.
4. On `COMPLETE`, the DispatcherAgent produces a `DispatchSummary { overview, assignments }` and emits `ScheduleCompleted`. The Schedule moves to `COMPLETED`.
5. After completion, the `FairnessMonitor` TimedAction evaluates the fairness ledger; on imbalance it records a `FairnessAlert`.
6. The operator can press **Pause new dispatches** at any time. The schedule finishes the in-flight step, then ends with `SchedulePausedOperator`. The Schedule moves to `PAUSED`.

A `WorkOrderSimulator` (TimedAction) drips a sample work order every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DispatcherAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains schedule ledger; reads fairness ledger. Produces `DispatchSummary` on completion. | `ScheduleWorkflow` | returns typed result to workflow |
| `RouteOptimizerAgent` | `AutonomousAgent` | Assigns a work order to the closest qualified available technician using seeded fixture data (`sample-data/technicians.jsonl`, `sample-data/work-orders.jsonl`). | `ScheduleWorkflow` | — |
| `AvailabilityAgent` | `AutonomousAgent` | Checks a technician's on-shift window and current assignment load from fixture data. | `ScheduleWorkflow` | — |
| `ScheduleWorkflow` | `Workflow` | Drives the plan → check-pause → propose → guardrail → dispatch → sanitize → record → fairness-check → decide loop, plus replan and pause branches. | `ScheduleEndpoint`, `WorkOrderConsumer` | `ScheduleEntity` |
| `ScheduleEntity` | `EventSourcedEntity` | Holds the schedule's lifecycle, schedule ledger, fairness ledger, and final dispatch summary. | `ScheduleWorkflow` | `ScheduleView` |
| `OperatorControlEntity` | `EventSourcedEntity` | Holds the operator pause flag. Single instance keyed by literal `"global"`. | `ScheduleEndpoint` (operator action) | `ScheduleWorkflow` (polls) |
| `WorkOrderQueue` | `EventSourcedEntity` | Audit log of submitted work orders. | `ScheduleEndpoint`, `WorkOrderSimulator` | `WorkOrderConsumer` |
| `ScheduleView` | `View` | List-of-schedules read model for the UI. | `ScheduleEntity` events | `ScheduleEndpoint` |
| `WorkOrderConsumer` | `Consumer` | Subscribes to `WorkOrderQueue` events; starts a `ScheduleWorkflow` per submission. | `WorkOrderQueue` events | `ScheduleWorkflow` |
| `WorkOrderSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/work-order-prompts.jsonl` and enqueues it. | scheduler | `WorkOrderQueue` |
| `StalledScheduleMonitor` | `TimedAction` | Every 30 s, marks any schedule stuck in `DISPATCHING` past 5 minutes as `STALLED`. The workflow polls this and ends with `ScheduleFailedTimeout`. | scheduler | `ScheduleEntity` |
| `FairnessMonitor` | `TimedAction` | Every 5 minutes, queries `ScheduleView.getAllSchedules`, evaluates fairness ledger data across completed schedules, and records `FairnessAlert` events on `ScheduleEntity` when imbalance exceeds thresholds. | scheduler | `ScheduleEntity` |
| `ScheduleEndpoint` | `HttpEndpoint` | `/api/schedules/*` — submit, get, list, SSE, operator pause. | — | `ScheduleView`, `WorkOrderQueue`, `ScheduleEntity`, `OperatorControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record WorkOrderRequest(String description, String serviceAddress, String requiredSkill, String requestedBy) {}

record ScheduleLedger(
    List<String> knownOrders,
    List<String> unassignedSlots,
    List<String> plan,
    Optional<AssignmentDecision> currentDispatch
) {}

record AssignmentDecision(
    SpecialistKind specialist,
    String assignment,
    String rationale
) {}

record AssignmentResult(
    SpecialistKind specialist,
    String assignment,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record AssignmentEntry(
    int attempt,
    SpecialistKind specialist,
    String assignment,
    AssignmentVerdict verdict,
    String scrubbedResult,
    Optional<String> blocker,
    Instant recordedAt
) {}

record FairnessLedger(
    List<AssignmentEntry> entries,
    List<FairnessAlert> alerts
) {}

record FairnessAlert(
    String technicianId,
    String alertType,
    String detail,
    Instant detectedAt
) {}

record DispatchSummary(String overview, List<String> assignments, Instant producedAt) {}

record Schedule(
    String scheduleId,
    String description,
    String serviceAddress,
    String requiredSkill,
    ScheduleStatus status,
    Optional<ScheduleLedger> ledger,
    Optional<FairnessLedger> fairness,
    Optional<DispatchSummary> summary,
    Optional<String> failureReason,
    Optional<String> pauseReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SpecialistKind { ROUTE_OPTIMIZER, AVAILABILITY }
enum AssignmentVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, FAIRNESS_FLAGGED }
enum ScheduleStatus { PLANNING, DISPATCHING, COMPLETED, FAILED, PAUSED, STALLED }
```

### Events (`ScheduleEntity`)

`ScheduleCreated`, `SchedulePlanned`, `AssignmentDispatched`, `AssignmentBlocked`, `AssignmentRecorded`, `LedgerRevised`, `FairnessAlertRecorded`, `ScheduleCompleted`, `ScheduleFailed`, `SchedulePausedOperator`, `ScheduleFailedTimeout`.

### Events (`OperatorControlEntity`)

`PauseRequested`, `PauseCleared`.

### Events (`WorkOrderQueue`)

`WorkOrderSubmitted { scheduleId, description, serviceAddress, requiredSkill, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/schedules` — body `{ description, serviceAddress, requiredSkill, requestedBy? }` → `202 { scheduleId }`. Starts a workflow.
- `GET /api/schedules` — list all schedules. Optional `?status=...`.
- `GET /api/schedules/{id}` — one schedule (full ledgers + summary).
- `GET /api/schedules/sse` — server-sent events stream of every schedule change.
- `POST /api/control/pause` — body `{ reason }` → `200`. Sets the operator pause flag.
- `POST /api/control/resume` — `200`. Clears the operator pause flag.
- `GET /api/control` — `{ paused, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Scheduling Operations Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a work order, operator pause/resume control, live list of schedules with status pills, expand-row to see the schedule ledger, the fairness ledger entries, and the final dispatch summary.

Browser title: `<title>Akka Sample: Scheduling Operations Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `DispatcherAgent`): every `AssignmentDecision` is checked against (a) the specialist allow-list, (b) technician shift-window constraints (no assignments outside the technician's declared on-shift hours), (c) qualification matching (requiredSkill must appear in the technician's skills list), (d) geographic feasibility (travel time estimate must not exceed the remaining shift time). Blocking. Failure → `AssignmentBlocked` entry + replan request.
- **E1 — periodic fairness watch** (`eval-periodic`, flavor `drift-fairness-watch`): `FairnessMonitor` runs every 5 minutes and evaluates per-technician assignment counts and travel loads across recent completed schedules. If any technician's assignment share exceeds 1.5× the fleet average, or if the same technician has been unassigned for > 3 consecutive scheduling cycles, a `FairnessAlert` is recorded. Non-blocking — the monitor does not stop in-progress schedules.
- **HO1 — operator pause control** (`hotl`, flavor `deployer-runtime-monitoring`): `OperatorControlEntity` (single instance keyed by `"global"`) holds the pause flag. The App UI shows Pause / Resume buttons backed by `POST /api/control/pause` and `POST /api/control/resume`. Every `ScheduleWorkflow` polls the flag in its `checkPauseStep` at the top of each loop iteration. On `paused=true` the workflow finishes the in-flight step, then emits `SchedulePausedOperator`.
- **S1 — credential sanitizer** (`sanitizer`, flavor `secret`): every `AssignmentResult.content` is scrubbed by a deterministic redactor matching AWS access keys, API tokens, JWTs, generic bearer tokens, and high-entropy strings ≥ 32 characters with Shannon entropy > 4.5 bits/char. Replacements are tagged (e.g., `[REDACTED:aws-access-key]`). Scrubbed result lands on the `AssignmentEntry` and is what the DispatcherAgent sees on the next loop tick.

## 9. Agent prompts

- `DispatcherAgent` → `prompts/dispatcher.md`. Maintains both ledgers; decides next step.
- `RouteOptimizerAgent` → `prompts/route-optimizer.md`. Returns technician-assignment results from fixtures.
- `AvailabilityAgent` → `prompts/availability.md`. Returns on-shift availability assessments.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Assign the next plumbing work order to the best available technician in Zone 3." Schedule progresses `PLANNING → DISPATCHING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a schedule ledger with a non-empty plan, a fairness ledger with 2–6 entries spanning at least ROUTE_OPTIMIZER and AVAILABILITY specialists, and a non-empty `DispatchSummary`.
2. **J2** — Submit a work order that would assign a technician outside their shift window. The guardrail blocks the dispatch; the dispatcher replans; the schedule either completes via a different technician or fails after the replan budget is exhausted.
3. **J3** — Submit a work order and click **Pause new dispatches** while it is `DISPATCHING`. The in-flight step finishes; no further dispatches occur; the schedule ends in `PAUSED`.
4. **J4** — A fixture technician record contains a credential-shaped string. The sanitizer scrubs it before it reaches the dispatcher's next prompt; the `AssignmentEntry.scrubbedResult` does not contain the original credential.
5. **J5** — After several schedules complete, `FairnessMonitor` detects one technician with 2× the fleet average assignments; a `FairnessAlert` entry appears in the fairness ledger; the operator dashboard shows the alert.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named field-service-dispatcher demonstrating the
planner-executor × ops-automation cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-ops-automation-field-service-dispatcher. Java
package io.akka.samples.schedulingoperationsagent. Akka 3.6.0. HTTP port 9658.

Components to wire (exactly):
- 3 AutonomousAgents:
  * DispatcherAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN_SCHEDULE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE_ASSIGNMENT).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_SUMMARY).maxIterationsPerTask(2)).
    System prompt from prompts/dispatcher.md. PLAN_SCHEDULE returns ScheduleLedger.
    DECIDE_ASSIGNMENT returns a NextStep tagged union (Continue(AssignmentDecision) |
    Replan(revisedLedger) | Complete(DispatchSummary stub) | Fail(reason)).
    COMPOSE_SUMMARY returns DispatchSummary.
  * RouteOptimizerAgent — capability(TaskAcceptance.of(FIND_TECHNICIAN).maxIterationsPerTask(2)).
    Prompt from prompts/route-optimizer.md. Returns AssignmentResult.
  * AvailabilityAgent — capability(TaskAcceptance.of(CHECK_AVAILABILITY).maxIterationsPerTask(2)).
    Prompt from prompts/availability.md. Returns AssignmentResult.

- 1 Workflow ScheduleWorkflow with steps:
  planStep -> [loop entry] checkPauseStep -> proposeStep -> guardrailStep ->
  dispatchStep -> sanitizeStep -> recordStep -> fairnessCheckStep -> decideStep
  -> [back to checkPauseStep or to completeStep / failStep / pausedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120) (covers any specialist call), decideStep ofSeconds(45),
    completeStep ofSeconds(60). defaultStepRecovery(maxRetries(2).failoverTo
    (ScheduleWorkflow::error)).
  checkPauseStep reads OperatorControlEntity.get; on paused=true transitions to
  pausedStep (emits SchedulePausedOperator on ScheduleEntity).
  guardrailStep runs the deterministic vetter over AssignmentDecision; on reject
  records an AssignmentBlocked entry via ScheduleEntity.recordBlock(assignment, reason)
  and loops back to proposeStep.
  dispatchStep uses switch on AssignmentDecision.specialist to call the
  matching specialist agent via forAutonomousAgent(...).runSingleTask(...)
  then forTask(scheduleId).result(...).
  sanitizeStep applies CredentialScrubber.scrub to the AssignmentResult.content.
  recordStep calls ScheduleEntity.recordAssignment(entry).
  fairnessCheckStep does a lightweight in-flight check — if the same technicianId
  has appeared in > 3 consecutive OK entries, records an inline FairnessAlert.
  decideStep calls forAutonomousAgent(DispatcherAgent.class, DECIDE_ASSIGNMENT);
  on Continue or Replan loops; on Complete transitions to composeSummaryStep
  -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity ScheduleEntity holding Schedule state. emptyState() returns
  Schedule.initial("", null) with no commandContext() reference. Commands:
  createSchedule, recordPlan, recordDispatch, recordBlock, recordAssignment,
  reviseLedger, recordFairnessAlert, completeSchedule, failSchedule, pauseOperator,
  timeoutFail, getSchedule. Events as listed in SPEC §5.

- 1 EventSourcedEntity OperatorControlEntity keyed by literal "global". State
  OperatorControl{boolean paused, Optional<String> reason, Optional<Instant>
  pausedAt}. Commands: requestPause(reason), clearPause, get. Events:
  PauseRequested, PauseCleared.

- 1 EventSourcedEntity WorkOrderQueue with command enqueueWorkOrder(scheduleId,
  description, serviceAddress, requiredSkill, requestedBy) emitting WorkOrderSubmitted.

- 1 View ScheduleView with row type ScheduleRow (mirror of Schedule minus heavy
  fairness payload — truncate to last 3 assignment entries plus counts; the UI
  fetches the full schedule by id on click). Table updater consumes ScheduleEntity
  events. ONE query getAllSchedules SELECT * AS schedules FROM schedule_view.
  No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer WorkOrderConsumer subscribed to WorkOrderQueue events; on
  WorkOrderSubmitted starts a ScheduleWorkflow with scheduleId as the workflow id.

- 3 TimedActions:
  * WorkOrderSimulator — every 90s, reads next line from
    src/main/resources/sample-events/work-order-prompts.jsonl and calls
    WorkOrderQueue.enqueueWorkOrder.
  * StalledScheduleMonitor — every 30s, queries ScheduleView.getAllSchedules,
    filters DISPATCHING schedules whose createdAt is older than 5 minutes, calls
    ScheduleEntity.timeoutFail; ScheduleWorkflow polls ScheduleEntity.getSchedule
    in its decideStep and exits when status == STALLED.
  * FairnessMonitor — every 5 minutes, queries ScheduleView.getAllSchedules for
    recently COMPLETED schedules, aggregates assignment counts per technicianId
    from their fairness ledgers, calls ScheduleEntity.recordFairnessAlert when
    imbalance thresholds are exceeded.

- 2 HttpEndpoints:
  * ScheduleEndpoint at /api with POST /schedules, GET /schedules (filters
    client-side from getAllSchedules), GET /schedules/{id}, GET /schedules/sse,
    POST /control/pause, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- DispatcherTasks.java declaring three Task<R> constants: PLAN_SCHEDULE
  (resultConformsTo ScheduleLedger), DECIDE_ASSIGNMENT (NextStep),
  COMPOSE_SUMMARY (DispatchSummary).
- SpecialistTasks.java declaring two Task<R> constants: FIND_TECHNICIAN,
  CHECK_AVAILABILITY (both resultConformsTo AssignmentResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface with
  permits Continue, Replan, Complete, Fail (each carrying its own payload —
  Continue with AssignmentDecision, Replan with revisedLedger, Complete with
  DispatchSummary stub, Fail with failureReason).
- application/CredentialScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36}, JWT
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens >= 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token],
  [REDACTED:jwt], [REDACTED:openai-key], [REDACTED:bearer-token],
  [REDACTED:high-entropy].
- application/AssignmentGuardrail.java — deterministic vetter. Reject if the
  specialist is not ROUTE_OPTIMIZER/AVAILABILITY, if an assignment references a
  technicianId not in the fixture roster, if the proposed assignment time falls
  outside the technician's on-shift window (from sample-data/technicians.jsonl),
  if the technician's requiredSkill list does not contain the work order's
  requiredSkill.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9658 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/work-order-prompts.jsonl with 8 canned
  work-order prompts covering plumbing, electrical, HVAC, and general maintenance.
- src/main/resources/sample-data/technicians.jsonl — 10 technician fixture
  records (id, name, skills, zone, shiftStart, shiftEnd, currentAssignments).
  One record's notes field contains an AKIA-shaped key fragment for the J4
  acceptance test.
- src/main/resources/sample-data/work-orders.jsonl — 12 canned work order
  fixtures (id, description, serviceAddress, zone, requiredSkill, estimatedHours).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 4 controls (G1, E1, HO1, S1)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls, and
  compliance.capabilities; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/dispatcher.md, prompts/route-optimizer.md, prompts/availability.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Scheduling Operations
  Agent", one-line pitch, prerequisites (including the integration form's
  host-software requirement: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator pause/resume control + live list with
  status pills and expand-on-click for ledgers and summary). Browser title
  exactly: <title>Akka Sample: Scheduling Operations Agent</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class
  name and the Task<R> id. Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent:
  dispatcher.json, route-optimizer.json, availability.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's
  typed return shape.
- Per-agent mock-response shapes:
    dispatcher.json — three lists keyed by task id:
      "PLAN_SCHEDULE" -> 4-6 ScheduleLedger entries spanning route and
      availability steps.
      "DECIDE_ASSIGNMENT" -> 4-6 NextStep entries covering Continue (with
      AssignmentDecision across both specialists), Replan, Complete, Fail.
      "COMPOSE_SUMMARY" -> 4-6 DispatchSummary entries with 60-120 word
      overviews and 2-4 assignment bullets.
    route-optimizer.json — 6 AssignmentResult entries, ok=true, content
      fields are technician assignment results (technicianId, zone, ETA).
    availability.json — 6 AssignmentResult entries; one entry's content
      must include the literal substring "AKIAIOSFODNN7EXAMPLE" so the J4
      sanitizer test fires.
- A MockModelProvider.seedFor(scheduleId) helper makes selection
  deterministic per schedule id so the same schedule in dev produces the
  same output across restarts.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on every agent-calling step.
- Lesson 6: Optional<T> for every nullable field on View row record and on
  the Schedule entity state.
- Lesson 7: AutonomousAgent requires companion DispatcherTasks.java and
  SpecialistTasks.java.
- Lesson 8: model-name values verified against the provider's current
  lineup: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: HTTP port 9658 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or metadata.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND themeVariables (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
